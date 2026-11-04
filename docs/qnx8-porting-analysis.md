# Pi → QNX 8.0 移植可行性分析

> 分析日期：2026-08（基于 GitHub 公开仓库证据）；代码增量核对至 2026-11-04，上游 `e88525506`（v1.5.0）。外部 QNX 运行时与工具链结论未在本轮重新验证。
> 范围：将 pi monorepo（`packages/coding-agent` 主 CLI/SDK）移植到 QNX Neutrino 8.0（x86_64 与 aarch64le 均适用）

## 结论

**可行**。核心阻塞（Node.js 运行时）已被 QNX 官方解决：QNX 8.0 通过 apk 提供 Node.js 24.14.1 LTS。v1.5.0 起 coding-agent 不再依赖 `chord`，其安装路径中已没有 esbuild（随包 install-lock 中 0 处 esbuild/chord），普通 `npm install` 即可完成安装，**不再需要 `--ignore-scripts`**。主要剩余工作量为 ripgrep/fd 两个外部工具的 QNX 构建，以及实际冒烟验证。

### 架构支持（aarch64le）

- QNX 8.0 以 aarch64 为一等目标（官方 BSP：`qnx/bsp_raspberrypi-bcm2712-rpi5`；QSTI 提供 QEMU aarch64 与 RPi5 镜像），仅 little-endian
- nodejs APKBUILD `arch="all"` → 所有架构可构建；aports 构建方式为 **QNX target 本机构建**（官方 codelab：QEMU 或 RPi5 上跑 `abuild`，产物输出 `~/packages/extra/<arch>/`），RPi5 上构建即产出 aarch64 apk
- qnx-ports/node `configure.py`：`valid_arch` 含 `arm64`，`maglev_enabled_architectures = ('x64', 'arm', 'arm64')` → V8 arm64 后端已启用
- rg/fd 交叉编译目标（QNX SDP 8.0）：Rust 官方 `aarch64-unknown-qnx`（x86_64 为 `x86_64-pc-qnx`）。注意 `*-nto-qnx710` 目标属于 QNX SDP 7.1，不适用于 8.0
- 佐证：aports 的 AI 生态包（llama.cpp #398、whisper.cpp #539、ncnn #543、pytorch #446）面向 aarch64 嵌入式/边缘场景，构建体系在 aarch64 上运行良好

## 证据链（GitHub 搜索）

### 1. QNX 官方 Node.js 24.14.1 LTS

- 仓库：[qnx-ports/aports](https://github.com/qnx-ports/aports)，PR [#354 extra/nodejs: new](https://github.com/qnx-ports/aports/pull/354)，2026-04-20 合并，维护者 Aaron Bassett (QNX)
- 安装：`sudo apk add nodejs npm`
- `APKBUILD` 关键构建参数：
  - `--shared-sqlite` → **`node:sqlite`（`DatabaseSync`）可用**，pi 的 SQLite 存储后端可用（实验性 `pi-durable` 的 `packages/durable/src/storage/sqlite/node.ts`；主线 coding-agent 使用自带 JSONL 会话管理，不依赖 SQLite）
  - `--shared-libuv --shared-openssl --shared-icu --shared-zlib` 等系统库
  - 版本策略：仅更新 LTS（`pkgver=24.14.1`）
- 配套源码 fork：[qnx-ports/node](https://github.com/qnx-ports/node)，tag `qnx-v24.14.1`，含 QNX 专用补丁：
  - `v8-int64-lowering-reducer.patch`
  - `v8-no-static-zlib.patch`
  - `node_sea.patch` / `node_snapshotable.patch`
  - `remove-unused-openssl-config.patch`、`ninja-patch`
- `configure.py` 的 `valid_os` 含 `qnx`，`common.gypi` 有 `OS=="qnx"` 分支 → QNX 是一等构建目标
- `node.gyp`：`NODE_PLATFORM="<(OS)"` → **`process.platform` 返回 `"qnx"`**（非 "linux"）

### 2. 官方概念验证：AI 编码助手跑在 QNX 上

- 仓库：[qnx/claude-code-qnx](https://github.com/qnx/claude-code-qnx)（QNX 官方 org，2026-06）
- 在 QNX 8.0 上运行 Claude Code：用 Node.js 替换 Bun 运行时（Bun 未移植到 QNX），Bun API shim + JS bundle 提取
- 与 pi 同类应用（AI 编程助手 + TUI + 子进程），证明该路线官方已走通
- 前置条件：QNX 8.0 + Node.js 18+（with npm）+ 网络

### 3. 配套工具（qnx-ports/aports）

| 包 | 状态 |
|----|------|
| bash 5.3 | ✅ 有（`provides /bin/sh`） |
| git | ✅ 有（`qnx-build.patch`） |
| nodejs 24.14.1 | ✅ 有 |
| ripgrep (rg) | ❌ 无 |
| fd | ❌ 无 |
| llama.cpp | ✅ 有（PR #398）→ pi 内置 llama 扩展可用 |

## 可行性矩阵

| 项 | 结论 | 说明 |
|----|------|------|
| 运行时 | ✅ | Node 24.14.1 LTS ≥ pi 要求 `>=22.19.0`；`node:sqlite`、`worker_threads`、`AbortSignal.any/timeout`、UDS (`AF_UNIX`)、信号/进程组均可用 |
| `process.platform` | ✅ 有利 | 返回 `"qnx"` → pi 的 win32/linux 分支都不误命中；`package-manager.ts` 的 `/proc/self/environ` 读取、footer-data-provider 的 WSL 检测等 `platform === "linux"` 门控逻辑被跳过 |
| npm 安装 | ✅ | v1.5.0 的运行时依赖树只有 `pi-agent-core`、`pi-ai`、`pi-codemode`、`pi-mcp`、`pi-tui` 及纯 JS/WASM 包（`quickjs-wasi`、`@silvia-odwyer/photon-node`、`typebox`、`marked`、`lovely-mermaid`、`cross-spawn` 等）；随包 `install-lock` 共 123 项，**无 esbuild、无 chord**。仍带 install script 的只有 `@google/genai`（`preinstall: echo 'preinstall: no-op'`，`prepare` 只在 git 依赖时运行）与 `protobufjs`（postinstall 打印赞助信息）——均为纯 JS，QNX 可执行，不阻塞安装。`pi-tui` 的剪贴板/修饰键原生 helper 以预编译 `.node` 随包分发（`native/{darwin,win32,linux}/prebuilds`），不是依赖也不在安装期编译。`--ignore-scripts` 仍可用作加固，但不再是必需 |
| 工具参数校验 | ✅ | v1.5.0 起工具参数由 `pi-schema` 校验：零依赖、不使用 `eval`，schema 编译为普通闭包；无 JIT/CSP 特权要求，QNX 上无额外依赖（`typebox` 仍作为 coding-agent 依赖保留，用于配置 schema） |
| bash 工具 | ✅ | aports 有 bash 5.3；`shell.ts` 路径 `/bin/bash` → PATH bash → `sh` 兜底成立 |
| git | ✅ | aports 有 git；footer 分支显示可用 |
| grep 工具 | ❌ 硬缺口 | 需要 `rg`；pi 的自动下载（`tools-manager.ts`）只支持 darwin/linux/win32 资产 |
| find 工具 | ⚠️ 缺口 | 需要 `fd`（有 `systemBinaryNames` 可先用系统命令） |
| TUI | ⚠️ 受限 | 全屏布局自 v1.0 起为默认（`tuiMode: "fullscreen"`：备用屏布局、滚轮加速、OSC 7501 状态上报），需终端支持备用屏与光标定位，可经设置切回内联模式；kitty keyboard protocol、bracketed paste、终端图片依赖连接的终端模拟器（SSH 登录场景可用）；修饰键 native helper（`native-platform.ts`）平台门控仅 darwin/win32 → QNX 下 `isNativeModifierPressed` 恒 false，降级不崩溃 |
| 剪贴板 | ⚠️ 降级 | `clipboard.ts`：native（pi-tui，无 QNX prebuild）→ 平台命令（xclip/xsel/wl-copy/pbcopy/clip）；QNX 无 X11/Wayland 工具，**SSH 远程会话下 `isRemoteSession()` 触发 OSC 52**，本地控制台直接抛 `Clipboard unavailable`；read 返回 null |
| headless 模式 | ✅ | `--mode rpc / print / json` 不依赖 TUI |
| OAuth | ⚠️ 需验证 | 无浏览器环境需走 device code 流程；Copilot/Radius/Kimi 已有实现（`packages/ai/src/auth/oauth/device-code.ts`，github-copilot 轮询带 429 重试），其余 provider 逐个确认 |

## 剩余缺口与解决路径

### ripgrep / fd（硬缺口）

**源码位置**：

| 工具 | 上游源码 | 本地副本 |
|------|----------|----------|
| ripgrep (rg) | https://github.com/BurntSushi/ripgrep | https://github.com/jefby/ripgrep（fork，master 分支） |
| fd | https://github.com/sharkdp/fd | https://github.com/jefby/fd（fork，master 分支） |

（pi 的 `tools-manager.ts` 硬编码了上游仓库：`repo: "BurntSushi/ripgrep"` / `repo: "sharkdp/fd"`）

解决路径（三选一）：

1. **Rust 交叉编译**（目标三元组与 QNX 版本对应关系）：

   | 目标 | QNX 版本 |
   |------|----------|
   | `aarch64-unknown-qnx` / `x86_64-pc-qnx` | **QNX SDP 8.0+**（`target_os="qnx"`，仅 io-sock 网络栈） |
   | `aarch64-unknown-nto-qnx710` / `x86_64-pc-nto-qnx710` | QNX SDP 7.1（io-pkt，**不适用 8.0**） |

   QNX 8.0 支持由 rust-lang/rust 2026-07 合并（PR #158449 改名、#158697 libstd 修复，需 libc-0.2 backport + cc-rs #1775），完整 std 标记为进行中（QEMU 上已验证 std 基本可用）。需要较新 Rust（含上述 PR，建议 nightly 或最新 stable）以及 QNX SDP 8.0 + `source qnxsdp-env.sh`（`qcc` 在 PATH）：
   ```bash
   rustup target add aarch64-unknown-qnx        # 或 x86_64-pc-qnx
   source /path/to/qnxsdp-env.sh                # 初始化 QNX SDP 8.0，qcc 入 PATH
   cargo build --release --target aarch64-unknown-qnx
   ```
2. **提交 aports PR**：qnx-ports/aports 接受外部贡献者（`eleir9268`、`jscaff` 等的 PR 已被合并），只需 `@qnx-ports/aports-admin` 审核，无 CLA 门槛。**注意：aports 目前没有 Rust 工具链**（core/extra 均无 rust/cargo 包），而构建在 QNX target 本机跑 abuild → 直接提交 ripgrep/fd 会因缺少 `cargo` makedepend 无法构建。需先提交 `rust`/`cargo` 包（大工程）或先开 issue 询问维护者是否计划引入（维护者 Aaron Bassett 活跃）
3. **放 PATH 即可**：`getToolPath()` 依次查本地 tools 目录、系统 PATH，均未命中才由 `ensureTool()` 触发下载（`tools-manager.ts`）；rg/fd 存在于 PATH 即被直接使用，无需下载

注意：若 `process.platform === "qnx"`，`getAssetName()` 对未知平台返回 `null` → 自动下载被跳过，不会误下载 linux 二进制（`ensureTool()` 捕获后仅给出 warning，grep/find 功能降级不崩溃）。离线 QNX 环境可设 `PI_OFFLINE=1` 直接跳过下载尝试，避免每工具最长 120s 的网络超时。

### esbuild（v1.5.0 起不再是安装阻塞）

- v1.1 及之前：`coding-agent` → `chord` → `esbuild`（带 postinstall），esbuild 的 `install.js` 按 `process.platform + arch + endianness` 查平台包表，`qnx arm64 LE` / `qnx x64 LE` 不在表中 → 抛 `Unsupported platform: qnx ...`，导致整个 `npm install` 失败，必须加 `--ignore-scripts`。
- v1.2.0 起（核对至 v1.5.0）：`coding-agent` 不再依赖 `chord`。chord 只被 `pi-durable` 使用，且把 `esbuild` 声明为**可选 peerDependency**（仅 `chord/plugins` 构建插件时加载），不再是普通依赖。随包 `packages/coding-agent/install-lock/package-lock.json` 中 esbuild、chord 均为 0 处。
- 因此 QNX 上 `npm install -g @earendil-works/pi-coding-agent` 直接可用。只有两种情况仍需要 esbuild：从源码开发 monorepo（根 `devDependencies`），或使用 `chord/plugins` 构建插件——此时可交叉编译 Go 版 esbuild，并用 `ESBUILD_BINARY_PATH` 指向该二进制（esbuild 的 install.js 与运行时 JS API 都支持该变量）。

## 落地步骤

1. QNX 8.0 配置 apk 源，`sudo apk add nodejs npm bash git`
2. 准备 rg/fd（交叉编译或 aports PR）
3. `npm install -g @earendil-works/pi-coding-agent`（v1.5.0 起无需 `--ignore-scripts`）→ `pi --version` 冒烟
4. 先跑 headless（`pi -p "..."`），再在 SSH 终端验证交互模式
5. 处理 OAuth（无浏览器时验证 device code 流程）
6. 可选用 Node 分发版（`dist/bundle/cli.js`，npm `pi` bin 入口）而非 Bun 二进制

## 参考链接

- https://github.com/qnx-ports/aports （APKBUILD: extra/nodejs, core/bash, core/git）
- https://github.com/qnx-ports/aports/pull/354 （nodejs aport）
- https://github.com/qnx-ports/node （QNX Node fork，tag qnx-v24.14.1）
- https://github.com/qnx/claude-code-qnx （Claude Code on QNX 官方示例）
- https://github.com/qnx-ports/aports/pull/398 （llama.cpp aport）

## 备注

- 初次分析（未搜索 GitHub 前）曾判断"Node 需自行移植、整体不建议"，该结论已被官方 aports 证据推翻，本文档为修正版。
- Bun 二进制分发版不适用 QNX（Bun 未移植），使用 Node 分发。

## 上游更新记录（0.85.1 → 2026-11-04）

截至 2026-09-15 的上一轮记录：`v0.85.1`（2026-09-05 发布）之后有约 97 个未发布提交（版本号仍为 0.85.1），与 QNX 部署相关的要点如下；2026-09-17、2026-10-11 与 2026-11-04 增量另列于后文。

### 性能

- **ai `EventStream` 队列 O(n²) → O(1)**（#9055）：原来用 `Array.shift()` 出队（每次 O(n)），改为双栈 `FifoQueue`（均摊 O(1)）；修复工具输出很大时排空 extension 事件流的二次方 CPU 占用
- **agent JSONL fork 索引内存下降**：每个非空列表只记录"首个仍存活的 append 序号"，不再保留每个存活元素
- **agent 内存 fork 直接从 live state 构造**：复用同一 fork 策略流式读取，省去整体拷贝/重放
- **ai Fireworks deferred tool loading**（#9323）：按需加载工具 schema，降低每请求工具负载
- 已在 0.85.1 中：Node 分发改为 esbuild 单文件 bundle + jiti 延迟到扩展加载 + 罕见语法按需加载（启动优化）；全屏下 Alt+滚轮 5 倍速滚动（#9166）

### 其他（不影响本文档 QNX 结论）

- 剪贴板重构 #9163（`clipboard.ts`：native → 平台命令 → OSC 52），已同步更新本文档剪贴板行
- 仓库链接改为 `earendil-works/pi`（#9278）
- coding-agent 运行时依赖升级（chalk 6、undici 8.10.2 等，#9341）
- 大量 provider/模型修复（Bedrock 缓存计价、DeepSeek `deepseek-flash` 命名、Copilot GPT 走 Responses 等）

### 2026-09-17 增量核对（`e4c75a732` → `5a3a03a7f`）

本轮合并 7 个上游提交，未修改 QNX 平台分支、工具下载或剪贴板实现，因此不改变前述移植缺口：

- **Google / Vertex 思考级别**（`16235fd93`）：共享级别转换，依据模型的 `thinkingLevelMap` 与支持范围选择参数，避免发送不支持的级别。
- **Anthropic thinking 重放**（`1283afd0d`）：保留请求模型 ID，将不同的响应模型名称存入 `responseModel`，避免模型别名变化破坏后续思考内容重放。
- **瞬时错误重试**（`e5d18382a`、`e98f287ee`）：新增 Cloudflare HTTP 520 与 Azure 高峰容量错误识别，沿用既有重试策略，不引入平台依赖。
- **TUI 与评测**（#9705、#9706）：`InteractiveMode` 支持注入终端实现；新增上下文 footer 扩展评测；评测从会话消息校验实际系统提示词，并在运行后校验失败时保留已收集的用量与会话诊断。新增的 `autoevals` 是私有 `packages/evals` 包的开发依赖，不是 CLI 的新增运行时依赖。文档对照评测需要 Docker，不是 QNX CLI 部署的前置条件。
- 另一个提交仅更新贡献者批准名单。

> 注：以上均为未发布内容，版本号与行为可能在下一个 release 前变化。本轮仅核对代码与文档，未在 QNX 设备上执行冒烟验证。

### 2026-10-11 增量核对（`5a3a03a7f` → `e6b9a6af5`，v0.86.0 → v1.1.0）

本轮合并约 300 个上游提交，覆盖 v0.86.x、v0.87.x、v0.99.x、v1.0.x、v1.1.0 多个发布。逐项核对平台分支、工具下载与剪贴板实现后，**QNX 移植结论不变**；与本部署相关的要点如下。

**包结构变化（不影响 QNX 安装路径）**

- `pi-agent-core` 移除实验性 harness（会话树、`SessionStorage`、JSONL/SQLite 后端、跨会话搜索）；`packages/session-backends/sqlite-node` 包删除，SQLite 存储进入实验性 `pi-durable`（`packages/durable/src/storage/sqlite/`，`node:sqlite`）。
- coding-agent 新增运行时依赖 `pi-codemode`（quickjs-wasi 沙箱脚本工具）与 `pi-mcp`（MCP 客户端）；`pi-durable`、`pi-env`（SSH 远程执行）为独立/实验包，不进入稳定 CLI 依赖树。
- 新依赖均为纯 JS/WASM（lockfile 无 install script），esbuild 仍仅经 `chord` 传递——`--ignore-scripts` 结论不变（见上文 esbuild 缺口）。**该结论已被 2026-11-04 的 v1.5.0 核对推翻**：coding-agent 不再依赖 chord，安装无需 `--ignore-scripts`。
- npm shrinkwrap 移除，上游推荐 managed installer；npm 分发仍为 Node esbuild 单文件 bundle，安装方式不变。

**平台相关行为**

- 全屏布局成为默认 TUI 模式（`tuiMode: "fullscreen"`，备用屏 + OSC 7501 状态上报）：SSH 终端需支持备用屏；可经设置切回内联模式。
- 内置工具新增 `powershell`（复用 bash 操作抽象，Windows 场景）；`DEFAULT_TOOL_NAMES` 仍为 `[read, bash, edit, write]`；`tools-manager.ts` 平台门控不变——`process.platform === "qnx"` 时 `getAssetName()` 返回 null，自动下载仍被跳过，rg/fd 放 PATH 即可。
- 剪贴板新增 Android/Termux 读取与 X11 广播图片能力校验；QNX 路径（OSC 52 fallback / 本地控制台不可用）结论不变。
- 9 月 17 日的 6 个提交（事件处理器取消订阅、强制系统提示词不落盘、信号终止的 shell 命令报失败、Vercel AI Gateway thinking 保留、Anthropic thinking 丢弃提示缩短）均无平台依赖。
- llama 内置扩展仍在（aports llama.cpp 可用的结论维持）；Node 引擎要求仍为 `>=22.19.0`（QNX apk 的 Node 24.14.1 满足）。

**OAuth（无浏览器场景）**

- 新增 Anthropic copy-code 登录（#10194）、workload identity federation（#10242）、MCP 服务器 OAuth（动态客户端注册、RFC 9207 `iss` 校验、step-up 授权、按服务器存储凭证）；无浏览器时仍以 device code 流程为主，逐 provider 验证的注意事项不变。

### 2026-11-04 增量核对（`e6b9a6af5` → `e88525506`，v1.1.0 → v1.5.0）

本轮合并约 650 个上游提交，覆盖 v1.2.0、v1.3.0、v1.4.0、v1.5.0 四个发布。**esbuild 安装阻塞已消除**，其余 QNX 结论不变。

**关键变化：安装不再需要 `--ignore-scripts`**

- `coding-agent` 不再依赖 `chord`：运行时依赖仅 `pi-agent-core`、`pi-ai`、`pi-codemode`、`pi-mcp`、`pi-tui` 加纯 JS/WASM 包。
- 随包 `install-lock`（123 项）中 esbuild、chord 均为 0 处；esbuild 现在只是 monorepo 的 devDependency 与 `chord` 的可选 peerDependency（仅 `chord/plugins` 构建插件时使用）。
- 仍带 install script 的两个包都是无害 JS：`@google/genai`（preinstall 为 `echo` no-op）、`protobufjs`（postinstall 打印赞助信息）。
- `pi-tui` 的原生 helper 仍以预编译 `.node` 随包分发（darwin/win32/linux），安装期不编译，QNX 上加载失败后降级。

**包结构与应用层**

- v1.2.0 删除实验性 `packages/server`、`packages/client`、`packages/protocol` 及 `coding-agent/src/experimental` 的客户端/服务器栈；CLI 不再涉及 Unix socket 服务与 CBOR 线协议。
- v1.5.0 新增 `packages/schema`（`pi-schema`）：零依赖、不使用 `eval` 的 JSON Schema 校验器，工具参数统一由它校验。对 QNX 无额外要求（纯 JS 闭包，无 JIT 假设）。
- 破坏性变更（影响扩展作者，不影响 QNX）：`pi-ai` 不再 re-export TypeBox，`typebox` 不再是 `pi-ai` 依赖；`Type.Refine()`、`Type.BigInt()`、codec、`Format.Set()` 等非 JSON Schema 特性对工具参数不再生效。`typebox` 仍是 `coding-agent` 自身配置 schema 的依赖。

**平台相关行为**

- 工具下载门控未变：`tools-manager.ts` 的 `getReleaseTarget()` 只映射 darwin/linux/win32，`process.platform === "qnx"` 时返回 undefined → 跳过自动下载并提示用系统包管理器安装；rg/fd 放入 PATH 即可（`PI_OFFLINE=1` 可避免超时）。
- 剪贴板新增 Termux、WSL（`clip.exe` 文件中转）与 OSC 52 强制开关（`PI_CLIPBOARD_OSC52=1`）；QNX 路径不变——SSH 会话走 OSC 52，本地控制台读取返回 null、写入抛 `Clipboard unavailable`。
- 修饰键 native helper 平台门控仍为 darwin/win32；QNX 上 `isNativeModifierPressed` 恒 false，降级不崩溃。
- TUI 全屏默认（`tuiMode: "fullscreen"`）不变；`pi-tui` 依赖改为 `marked`、`marked-cjk-friendly`、`get-east-asian-width`，均为纯 JS。
- grep 工具改为由 ripgrep 直接输出 context 行，不再回读匹配文件（#9276）——QNX 上大文件匹配的内存压力降低。
- 图像处理新增 worker 线程准备路径（Photon WASM），Node 24 的 `worker_threads` 可用。
- Node 引擎要求仍为 `>=22.19.0`，QNX apk 的 Node 24.14.1 满足。

**其他（与 QNX 无关，但在同一批提交中）**

- Anthropic 消息线程（message threads beta，只发增量消息）、GPT-6.1 Terra、Qwen Token Plan Individual CN provider、`Usage.cost.totalFromProvider`、models.dev v2 模型目录。
- `pi-durable` 的 `Storage.commit()` 改为批量契约（`commit(batches, receive, context)`），`/tree` 分支实现为 durable fork——只影响实验包，不在 CLI 运行时路径上。

> 注：本轮同样只核对代码与文档，未在 QNX 设备上执行冒烟验证。安装方式的结论来自 `packages/coding-agent/install-lock/package-lock.json` 与各包 `package.json`，不是实机结果。
