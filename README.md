# MATLAB MCP Server 接入 DeepSeek Harness 实录

> **通用性**：第 2、5、6、7 节的流程对**任何 stdio 传输的 MCP 服务器**都适用，不限于 MATLAB。
> **窄带宽友好**：下载段内置断点续传，弱网环境可直接复用。

## ⚠️ 先读：版本敏感声明

**DSH 正在快速迭代，本文所有涉及 DSH 行为的结论只对下表这一组版本负责。**
换版本后请重点复核：第 4 节的配置字段、第 5 节的 `scrubbedParentEnv()` 净化规则、第 6 节的 `--dump-config` 行为。

| 组件 | 实测版本 |
|---|---|
| **DeepSeek Harness** | **0.1.5-rc.1** |
| `@deepseek-ai/dsh-mcp-client` | 0.1.5-rc.1 |
| `@deepseek-ai/dsh-subprocess` | 0.1.5-rc.1 |
| `@deepseek-ai/dsh-app-boot` | 0.1.5-rc.1 |
| `@modelcontextprotocol/sdk` | 1.30.0 |
| Node.js | v26.4.0 |
| MATLAB MCP Server | v0.13.0 |
| MATLAB | R2024b Update 9 |
| 操作系统 | Windows 11（Build 22631） |

查你自己的版本：

```powershell
dsh --version
# 各插件的版本（DSH 主程序与插件同版本号发布，正常情况下三者一致）
foreach ($p in 'dsh-mcp-client','dsh-subprocess','dsh-app-boot') {
    (Get-Content "$env:DSH_HOME\profiles\node_modules\@deepseek-ai\$p\package.json" -Raw | ConvertFrom-Json).version
}
```

> 📌 **DSH 主程序与其 `@deepseek-ai/*` 插件同版本号发布**，所以只要 `dsh --version` 与上表不符，
> 就应当把本文相关结论重新验证一遍——好在第 5 节给了你一个能一次验完的方法。

### 占位符约定

本文用环境变量和占位符代替具体路径，**照抄前请先确认这几项**：

| 占位符 | 含义 | 怎么取 |
|---|---|---|
| `%DSH_HOME%` | DSH 主目录 | DSH 会自动设置该环境变量，通常是 `%USERPROFILE%\.dsh`；PowerShell 里写 `$env:DSH_HOME` |
| `%USERPROFILE%` | Windows 用户目录 | 如 `C:\Users\你的名字`；PowerShell 里写 `$env:USERPROFILE` |
| `<MATLAB安装目录>` | MATLAB 安装根目录 | **不要带 `/bin` 后缀**，如 `D:\MATLAB\R2024b` |

> ⚠️ **重要**：`cordis.patch.yml` 里的路径是**字面值，DSH 不会展开环境变量**。
> 所以配置文件里必须填**绝对路径**（见第 4 节）；环境变量写法只适用于 PowerShell 脚本与正文说明。

---

## 0. 总览：七步流程

```
① 查版本要求 → ② 断点续传下载 → ③ 隔离目录安装 → ④ 协议预检
   → ⑤ overlay 校验 → ⑥ 落盘 + 复核 → ⑦ 端到端测试
```

**核心心法**：**在写配置文件之前，先用一个"和 DSH 完全一样的进程"把服务器跑通。**
第 ④ 步是整条链路里性价比最高的一环——它把"配置写错了"和"服务器本身有问题"这两类故障提前分离开。

---

## 1. 版本要求（先确认，别急着下载）

MATLAB MCP Server 对 MATLAB 的要求分三层，**只有第 1 条是硬门槛**：

| 能力 | 最低 MATLAB 版本 |
|---|---|
| 启动 MATLAB、读写代码、静态检查（基本功能） | **R2021a** |
| `--matlab-session-mode=existing` 连接已开启的会话 | **R2023a** |
| 纯文本 Live Code（`.m` 格式 live script） | **R2025a** |

其它要点：

- `matlab` 必须加入系统 **PATH**（或用 `--matlab-root` 显式指定，推荐后者）。
- 服务器本身**不需要任何 API Key / Token / 环境变量**——这是它比多数 MCP 服务器省事的地方。
- 二进制是 Go 单文件，**不需要 Node 或 Python 运行时**。

> 📌 **纯文本 Live Code 不影响普通 `.m` 脚本**。它的标记（`%[text]`、`%[appendix]` 等）全部以 `%` 开头，就是注释；在 R2024b 上跑这种文件不报错、代码照常执行，只是没有富文本渲染。所以 R2025a 这个门槛基本可以忽略。

---

## 2. 断点续传下载（窄带宽必备）

`Invoke-WebRequest` 不支持续传，**用系统自带的 `curl.exe`**（Windows 10+ 内置，`-C -` 从已有文件长度续传）：

```powershell
$url    = 'https://github.com/matlab/matlab-mcp-server/releases/latest/download/matlab-mcp-server-windows-x64.exe'
$out    = "$env:DSH_HOME\mcp-servers\matlab\matlab-mcp-server.exe"
$expect = 19232096                      # v0.13.0 的确切字节数，升级版本后需重新核对
$proxy  = 'http://127.0.0.1:8902'       # 无代理则删掉所有 --proxy 参数

New-Item -ItemType Directory -Force -Path (Split-Path $out) | Out-Null

for ($i = 1; $i -le 60; $i++) {
    $have = if (Test-Path $out) { (Get-Item $out).Length } else { 0 }
    if ($have -eq $expect) { break }
    if ($have -gt $expect) { Remove-Item $out -Force; $have = 0 }   # 超长说明下坏了，重来
    if ($have -eq 0 -and (Test-Path $out)) { Remove-Item $out -Force }

    Write-Host "[$i/60] $have / $expect bytes"
    & curl.exe -L -C - --proxy $proxy --connect-timeout 20 `
        --retry 8 --retry-delay 5 --retry-all-errors --max-time 7200 `
        -o $out $url
    Start-Sleep -Seconds 3
}

# 三重校验：字节数 + PE 魔数 + 哈希
if ((Get-Item $out).Length -eq $expect) {
    $fs = [System.IO.File]::OpenRead($out); $b = New-Object byte[] 2
    $fs.Read($b,0,2) | Out-Null; $fs.Close()
    "MAGIC : {0:X2} {1:X2}  (4D 5A = PE)" -f $b[0], $b[1]
    "SHA256: $((Get-FileHash $out -Algorithm SHA256).Hash)"
}
```

**为什么这样写**：

- `-C -` 放在**循环体内**，每次重试都重新读一次文件长度算偏移量——比依赖 `--retry` 自己续传更可靠。
- `--retry-all-errors`（需 curl ≥ 7.71）让连接被掐断也重试，不只是 HTTP 错误码。
- 校验三层都不能省：**字节数**防截断，**PE 魔数**防下成 HTML 错误页，**SHA256** 可对照 Release 页面。

> 实测 18.3 MB / ~100 KB/s 约 3 分钟，一次通过。若中途断线，重跑脚本即从断点继续。

---

## 3. 隔离目录安装

**绝不**装进全局环境或 Harness 安装目录：

```
%DSH_HOME%\mcp-servers\<server-name>\
```

本次即 `%DSH_HOME%\mcp-servers\matlab\matlab-mcp-server.exe`。
（Python 服务器用 venv、Node 服务器用 `npm install --prefix`，同理隔离。）

---

## 4. 写配置行

DSH 通过插件 `@deepseek-ai/dsh-mcp-client` 接入 MCP，**每个服务器 = 组合树中的一行**。
用户侧写在 `%DSH_HOME%\cordis.patch.yml`（全局，作用于所有 profile）。

本次追加的行（**下面三处路径请替换成你自己的绝对路径**）：

```yaml
- insert:
    - id: mcp-matlab
      name: '@deepseek-ai/dsh-mcp-client'
      config:
        serverName: matlab
        transport: stdio
        command: <DSH_HOME>/mcp-servers/matlab/matlab-mcp-server.exe
        args: ['--matlab-root=<MATLAB安装目录>', '--disable-telemetry=true']
        cwd: <DSH_HOME>/mcp-servers/matlab
        failOnStartupError: true
        toolCallTimeoutMs: 900000
```

实际填进去的形态类似：`C:/Users/<用户名>/.dsh/mcp-servers/matlab/matlab-mcp-server.exe`、
`--matlab-root=D:\MATLAB\R2024b`、`C:/Users/<用户名>/.dsh/mcp-servers/matlab`。

**几个关键决策**：

| 字段 | 取值理由 |
|---|---|
| `args: --matlab-root=...` | 显式指定比依赖 PATH 稳，也顺手规避"PATH 继承异常"这类问题；MATLAB 已在 PATH 上时可省略 |
| `--disable-telemetry=true` | 服务器默认向 MathWorks 上报匿名数据；窄带宽/离线环境建议关掉 |
| `failOnStartupError: true` | **调试期必开**——连不上直接激活失败，而不是静默变成 0 个工具 |
| `toolCallTimeoutMs: 900000` | ⚠️ **默认仅 60000ms**。MATLAB 默认**首次调用工具时才启动**，冷启动可能超 1 分钟，用默认值必超时 |
| 未设 `--matlab-display-mode` | 保持默认 `desktop`，会弹出 MATLAB 窗口，可手动接管 |

**三个语法 / 兼容坑**：

1. Windows 路径用**单引号**包裹（`'--matlab-root=D:\MATLAB\R2024b'`），单引号内反斜杠不转义；用双引号会踩转义问题。
2. `command` / `cwd` 等字段是**字面值**，写 `%DSH_HOME%` 或 `$env:DSH_HOME` **不会被展开**，必须填绝对路径。
3. **字段名随 DSH 版本漂移**。本文的 `serverName` / `transport` / `command` / `args` / `cwd` / `env` / `failOnStartupError` / `toolCallTimeoutMs` 均为 **0.1.5-rc.1** 实测。换版本后若该行加载失败，**先查你本机该版本插件自带的 README**，不要凭本文猜：

   ```powershell
   Get-Content "$env:DSH_HOME\profiles\node_modules\@deepseek-ai\dsh-mcp-client\README.md"
   ```

---

## 5. 协议预检 ⭐ 最有价值的一步

### 为什么不能只跑 `--help`

`--help` 只能证明"文件是个能执行的东西"，**证明不了它是个 MCP 服务器**。
真正的预检要**复刻 DSH 启动子进程的方式**，然后做一次真实 MCP 握手。

### 关键：env 必须用插件自己的净化逻辑

DSH 的 `@deepseek-ai/dsh-mcp-client` 构造子进程环境用的是：

```js
// 插件源码（@deepseek-ai/dsh-mcp-client/lib/index.js）里是这么调用的：
import { scrubbedParentEnv } from '@deepseek-ai/dsh-subprocess';

// 它的定义（@deepseek-ai/dsh-subprocess/lib/index.js）等价于下面两行规则：
const SENSITIVE_ENV_PATTERN = /KEY|PASSWORD|SECRET|TOKEN/i;
// 剔除「变量名匹配该正则」的项 + 剔除所有 DSH_* 前缀；其余整体继承
```

> ⚠️ 上面这个裸 `import` **只有在包能被解析时才行**——即脚本需位于 `%DSH_HOME%\profiles\` 这类存在
> node_modules 链的目录下，放到 `%TEMP%` 里跑会报 `ERR_MODULE_NOT_FOUND`。
> 下面给出的完整预检脚本**把这两条规则内联**了，因此放任何目录都能直接跑——**请用那一份**。

> 🔍 **换了 DSH 版本后请自查**——净化规则随版本漂移，**以你本机源码为准**：
> ```powershell
> Select-String -Path "$env:DSH_HOME\profiles\node_modules\@deepseek-ai\dsh-subprocess\lib\index.js" `
>               -Pattern "SENSITIVE_ENV_PATTERN =|DSH_ENV_PREFIX ="
> ```
> 输出若与上文的两行规则不一致，请以源码为准更新你的预检脚本。

⚠️ **它整体继承宿主环境，而不是用 MCP SDK 的默认白名单**——这一点很重要：

- **好消息**：`WINDIR` / `SystemRoot` / `PATH` / `MLM_LICENSE_FILE`（含 "LICENSE" 而非 "KEY"）全部幸存，MATLAB 能正常启动。
- **对比**：MCP SDK 的 `getDefaultEnvironment()` 白名单里**没有 `WINDIR`**，这正是官方 README 里 [Codex 踩的坑（issue #32）](https://github.com/matlab/matlab-mcp-server/issues/32)——**DSH 不会踩这个坑**。

### 预检脚本（Node ESM，DSH 自带 Node 可直接跑）

```js
// preflight.mjs —— 用与 DSH 完全相同的 command/args/cwd/env 做 MCP 握手
import { spawn } from 'node:child_process';
import process from 'node:process';

// ======== 只需确认这一项；其余路径从环境变量自动推导 ========
const MATLAB_ROOT = process.env.MATLAB_ROOT ?? String.raw`D:\MATLAB\R2024b`;  // ← 改成你的 MATLAB 安装目录

const DSH_HOME   = process.env.DSH_HOME;                  // DSH 自动设置，通常无需修改
const SERVER_DIR = `${DSH_HOME}/mcp-servers/matlab`;
const BIN        = `${SERVER_DIR}/matlab-mcp-server.exe`;
const ARGS       = [`--matlab-root=${MATLAB_ROOT}`, '--disable-telemetry=true'];
const CWD        = SERVER_DIR;

if (!DSH_HOME) { console.error('环境变量 DSH_HOME 未设置，请在 DSH 提供的 shell 中运行本脚本'); process.exit(1); }

const SENSITIVE_ENV_PATTERN = /KEY|PASSWORD|SECRET|TOKEN/i;
function scrubbedParentEnv() {
  const env = {};
  for (const [k, v] of Object.entries(process.env)) {
    if (v !== undefined && !SENSITIVE_ENV_PATTERN.test(k) && !k.toUpperCase().startsWith('DSH_')) env[k] = v;
  }
  return env;
}

const env = scrubbedParentEnv();
const child = spawn(BIN, ARGS, { cwd: CWD, env, stdio: ['pipe', 'pipe', 'pipe'] });

let buf = '', stderrBuf = '';
const pending = new Map();
child.stdout.on('data', (d) => {
  buf += d.toString();
  let i;
  while ((i = buf.indexOf('\n')) !== -1) {
    const line = buf.slice(0, i).trim(); buf = buf.slice(i + 1);
    if (!line) continue;
    let m; try { m = JSON.parse(line); } catch { continue; }
    if (m.id !== undefined && pending.has(m.id)) { pending.get(m.id)(m); pending.delete(m.id); }
  }
});
child.stderr.on('data', (d) => { stderrBuf += d.toString(); });

const send = (id, method, params, ms = 600000) => new Promise((res, rej) => {
  const t = setTimeout(() => rej(new Error(`timeout on ${method}`)), ms);
  pending.set(id, (m) => { clearTimeout(t); res(m); });
  child.stdin.write(JSON.stringify({ jsonrpc: '2.0', id, method, params }) + '\n');
});

try {
  const init = await send(1, 'initialize', {
    protocolVersion: '2025-06-18', capabilities: {},
    clientInfo: { name: 'preflight', version: '1.0.0' },
  }, 30000);
  if (init.error) throw new Error(JSON.stringify(init.error));
  console.log('serverInfo :', JSON.stringify(init.result.serverInfo));
  console.log('protocol   :', init.result.protocolVersion);

  child.stdin.write(JSON.stringify({ jsonrpc: '2.0', method: 'notifications/initialized', params: {} }) + '\n');

  const tools = await send(2, 'tools/list', {}, 30000);
  for (const t of tools.result.tools) console.log('  - mcp__matlab__' + t.name);

  // 再真调一个最轻的工具，验证端到端
  const call = await send(3, 'tools/call', {
    name: 'evaluate_matlab_code',
    arguments: { code: 'disp("OK"); disp(version)', project_path: process.env.USERPROFILE ?? CWD },
  });
  console.log('tool output:', call.result.content.map((c) => c.text).join('\n'));
  console.log(call.result.isError ? 'RESULT: TOOL ERROR' : 'RESULT: PASS');
  child.kill(); process.exit(call.result.isError ? 1 : 0);
} catch (e) {
  console.log('RESULT: FAILED -', e.message, stderrBuf.slice(-1500));
  child.kill(); process.exit(1);
}
```

**跑法**：`node preflight.mjs`

**它一次能同时证伪**：命令路径错、参数名错、传输协议不对、依赖缺失、后端起不来。

> 💡 **踩坑提醒**：`scrubbedParentEnv()` 返回的是**普通对象**，键名按 Windows 原样保留——
> 常见的是 `Path`、`windir`，**不是** `PATH`、`WINDIR`。
> 用 `env.PATH` 取值会读到 `undefined`，让你误报"变量缺失"。
> **探测环境变量时务必大小写不敏感**（Windows 本身查找是不敏感的，Go 的 `os.Getenv` 也不敏感，所以服务器没问题——有问题的只是你的探针）。

---

## 6. 安全写入协议（不可跳步）

配置写坏会让**整个 profile 启动失败**，且运行中的 Harness 会**静默保持上一个可用树**（错误只进日志）。所以：

```powershell
# ① 备份（无条件先做）
Copy-Item "$env:DSH_HOME\cordis.patch.yml" `
          "$env:DSH_HOME\cordis.patch.yml.bak-$(Get-Date -Format yyyyMMdd-HHmmss)"
```

```powershell
# ② 只把"增量行"写进临时文件（放 %TEMP%，别放被监听的 %DSH_HOME%），做隔离试组合
dsh --profile web --dump-config --patch "$env:TEMP\delta.yml"
```

**判定通过**：退出码 0 **且**新行出现 **且** 既有行一个没少（顺带证明旧文件是健康的）。
失败就改 delta 重试，**最多 2 次**；仍失败则停止，不要落盘。

```powershell
# ③ 合并落盘：把校验过的 delta 逐字节追加进真实文件
# ④ 复核：再跑一次不带 overlay 的命令，确认真实生效树
dsh --profile web --dump-config
```

> `--dump-config`（含 `--patch`）有两个盲区：
> 1. **不加载 `.env` 层**——它通过 ≠ `.env` 健康；
> 2. 会把 `!!js` 表达式**原样回显而不求值**——所以 dump 里看到 `!!js ...` 既不代表它坏了，也不代表它能在运行期正确求值。想验证 `!!js` 只能走真实启动路径。

---

## 7. 端到端验证

落盘后文件被**热监听（HMR）**，无需重启进程。两种确认方式：

1. **看工具表**：新工具会以 `mcp__<serverName>__<原名>` 出现。
2. **直接调用**（最硬的证据）：

```
mcp__matlab__evaluate_matlab_code  → 返回 "24.2.0.3212159 (R2024b) Update 9"
mcp__matlab__detect_matlab_toolboxes → 返回已装工具箱清单
mcp__matlab__run_matlab_file       → 执行 .m 文件，pwd 自动切到脚本所在目录
```

本次服务器暴露的完整能力：

| 类型 | 名称 |
|---|---|
| Tool | `detect_matlab_toolboxes` / `check_matlab_code` / `evaluate_matlab_code` / `run_matlab_file` / `run_matlab_test_file` |
| Resource | `guidelines://coding` / `guidelines://plain-text-live-code` |

---

## 8. 踩坑记录

### 8.1 全局 vs profile 作用域

`%DSH_HOME%\cordis.patch.yml` 是**全局**的，作用于**所有 profile**。
实测本机同时跑着 `web` 和 `dsh-tui` 两个 DSH 实例，落盘后**两边都热加载了**，各自起了一份服务器进程。

看到"进程数翻倍"别慌：**每个实例 = 主进程 + watchdog 子进程**。4 个进程 = 2 实例 × 2，正常。

### 8.2 多实例会各开一个 MATLAB

每个 MCP 服务器实例独立管理自己的 MATLAB 会话。若两个 DSH 实例都调用 MATLAB，会**各起一个 MATLAB（各约 1.7 GB 内存）**。

想共用一个 MATLAB，用 `--matlab-session-mode=existing`，并事先：

```powershell
./matlab-mcp-server --setup-matlab      # 安装 MATLAB MCP Server Toolbox 插件
```

然后在 MATLAB 命令窗口执行 `shareMATLABSession()`（可写进 `startup.m` 自动执行）。

> ⚠️ `existing` 模式下**不要**同时使用 `matlab-root` / `initial-working-folder` / `matlab-display-mode`，否则报错。
> 默认的 `auto` 模式则是"先找已共享的会话，找不到就自己起一个"，最省心。

### 8.3 `toolCallTimeoutMs` 默认值太小

默认 **60000ms**，而 MATLAB 首次工具调用才冷启动，很容易超时。
**装 MATLAB MCP 必须调大**（本次设 900000）。

### 8.4 `mcp_probe` 工具并非到处都有

一些教程会建议用 `mcp_probe` 探活。**它在部分 DSH 部署里并不存在**——别去工具表里白找，直接用手工协议预检（第 5 节）替代，验证得还更彻底。

### 8.5 Windows 环境变量名大小写

见第 5 节踩坑提醒。**这是本次唯一一次误报**，代价是差点误判成 DSH 的 bug。

---

## 9. 回滚

```powershell
# ① 还原配置（字节级）
Copy-Item "$env:DSH_HOME\cordis.patch.yml.bak-<时间戳>" "$env:DSH_HOME\cordis.patch.yml" -Force
# ② 删除安装目录
Remove-Item "$env:DSH_HOME\mcp-servers\matlab" -Recurse -Force
# ③ 复核
dsh --profile web --dump-config     # 退出码 0 且新增行已消失
```

备份文件建议**等实际用顺了再删**。

---

## 10. 本次结果一览

| 项 | 值 |
|---|---|
| 服务器 | MATLAB MCP Server **v0.13.0**（MathWorks 官方） |
| 二进制 | `%DSH_HOME%\mcp-servers\matlab\matlab-mcp-server.exe`，18.34 MB |
| SHA256 | `4E065398CF86E9D1D4D3E30CEC7FAF6F16491AB08128DA9A8336369166AD9917` |
| MATLAB | R2024b Update 9，含 Simulink / Audio / DSP System / Signal Processing |
| 配置 | `%DSH_HOME%\cordis.patch.yml` 新增 `mcp-matlab` 行 |
| 验证 | 协议握手 ✅ · 配置组合 ✅ · 落盘复核 ✅ · 端到端三工具 ✅ |

**总耗时**：约 15 分钟（含 3 分钟下载）。**无需任何凭据配置**——这是它最省事的地方。
