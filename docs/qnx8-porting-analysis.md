# Pi → QNX 8.0 移植可行性分析

> 分析日期：2026-08（基于 GitHub 公开仓库证据）；代码增量核对至 2026-09-26，上游 `d6af72e18`。外部 QNX 运行时与工具链结论未在本轮重新验证。
> 范围：将 pi monorepo（`packages/coding-agent` 主 CLI/SDK）移植到 QNX Neutrino 8.0（x86_64 与 aarch64le 均适用）

## 结论

**可行**。核心阻塞（Node.js 运行时）已被 QNX 官方解决：QNX 8.0 通过 apk 提供 Node.js 24.14.1 LTS。主要剩余工作量为 ripgrep/fd 两个外部工具的 QNX 构建、规避 esbuild 的 postinstall（安装加 `--ignore-scripts`），以及实际冒烟验证。

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
  - `--shared-sqlite` → **`node:sqlite`（`DatabaseSync`）可用**，pi 的 `session-backends/sqlite-node` 后端可用
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
| npm 安装 | ⚠️ 需 `--ignore-scripts` | Node 分发为 esbuild 单文件 bundle（bin → `dist/bundle/cli.js`，纯 ESM）；多数依赖纯 JS + WASM（`@silvia-odwyer/photon-node` 是 WASM；剪贴板/修饰键原生能力由 `@earendil-works/pi-tui` 随包预编译 `.node` 提供，仅 darwin/win32/linux-X11，安装不编译）。但新版 `@earendil-works/chord` 依赖 `esbuild`（带 postinstall，无 qnx 平台包）→ QNX 上直接 `npm install` 会因 esbuild postinstall 抛 `Unsupported platform: qnx ...` 失败，需 `npm install --ignore-scripts`（见下文 esbuild 缺口） |
| bash 工具 | ✅ | aports 有 bash 5.3；`shell.ts` 路径 `/bin/bash` → PATH bash → `sh` 兜底成立 |
| git | ✅ | aports 有 git；footer 分支显示可用 |
| grep 工具 | ❌ 硬缺口 | 需要 `rg`；pi 的自动下载（`tools-manager.ts`）只支持 darwin/linux/win32 资产 |
| find 工具 | ⚠️ 缺口 | 需要 `fd`（有 `systemBinaryNames` 可先用系统命令） |
| TUI | ⚠️ 受限 | kitty keyboard protocol、bracketed paste、终端图片依赖连接的终端模拟器（SSH 登录场景可用）；修饰键 native helper（`native-platform.ts`）平台门控仅 darwin/win32 → QNX 下 `isNativeModifierPressed` 恒 false，降级不崩溃 |
| 剪贴板 | ⚠️ 降级 | `clipboard.ts`：native（pi-tui，无 QNX prebuild）→ 平台命令（xclip/xsel/wl-copy/pbcopy/clip）→ OSC 52；X11 图片现要求 clipboard owner 宣告图像 target（未宣告的文本不再误判为图片）；QNX 无 X11/Wayland 工具，**SSH 远程会话下 `isRemoteSession()` 触发 OSC 52**，本地控制台直接抛 `Clipboard unavailable`；read 返回 null |
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

### esbuild（仅 experimental 功能受影响）

- coding-agent 自 0.85.1 起依赖 `@earendil-works/chord`，chord 又依赖 `esbuild`（用在 `packages/chord/src/node/bundle.ts`，即 `chord/bundler`、`chord/node` 入口）
- esbuild 0.28.2 带 postinstall（`hasInstallScript: true`），其 `install.js` 按 `process.platform + arch + endianness` 查平台包表；`qnx arm64 LE` / `qnx x64 LE` 不在 `knownUnixlikePackages` 中 → 直接抛 `Unsupported platform: qnx arm64 LE`，导致整个 `npm install` 失败
- 稳定 `pi` CLI 不加载 chord/esbuild：Node bundle 中 `@earendil-works/chord` 为 external，且只有 `src/experimental/**`（`PI_EXPERIMENTAL=1` 的插件打包 `bundleFacetPackage`、服务模式）会静态 import esbuild
- 解决：
  1. **安装加 `--ignore-scripts`**（推荐）：跳过 esbuild 的 postinstall，稳定 CLI 不受影响
  2. 若需 experimental 插件打包：esbuild 是 Go 程序，可交叉编译到 QNX，再用 `ESBUILD_BINARY_PATH` 指向该二进制（esbuild 的 install.js 与运行时的 JS API 均支持该环境变量覆盖）

## 落地步骤

1. QNX 8.0 配置 apk 源，`sudo apk add nodejs npm bash git`
2. 准备 rg/fd（交叉编译或 aports PR）
3. `npm install -g @earendil-works/pi-coding-agent --ignore-scripts`（见 esbuild 缺口）→ `pi --version` 冒烟
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

## 上游更新记录（0.85.1 → 2026-09-25）

截至 2026-09-15 的上一轮记录：`v0.85.1`（2026-09-05 发布）之后有约 97 个未发布提交（版本号仍为 0.85.1），与 QNX 部署相关的要点如下；2026-09-17 增量另列于后文。

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

### 2026-09-25 增量核对（`5a3a03a7f` → `5fd446ca1`，含 v0.87.0/v0.87.1）

本轮合并约 154 个上游提交并发布了 v0.87.0（2026-09-21）、v0.87.1（2026-09-22），与 QNX 部署相关的要点：

- **剪贴板**：X11 图片要求 owner 宣告图像 target；失败上报、未验证本地写入拒绝、OSC 52 headless 回退恢复 → 无变化结论（QNX 无 X11，SSH 路径本质不变）。
- **TUI**：Kitty 协议图片尺寸按宽高比失真量选择减少拉伸（#9957）；主题色支持 hex/OKLCH 值与 appearance 字段、`tui/src/colors.ts` 颜色助手 → 纯 JS 渲染逻辑，无影响。
- **HTML 导出**：新增隐藏消息开关（#10020）→ 无影响。
- **模型目录**：布局/索引校验/版本选择收敛到 `scripts/model-catalog-protocol.ts` 并与 pi.dev 共享；新增 GPT-6 Sol/Luna、Claude Opus 5.5、Grok 4.7、Copilot 模型 → 仅运行时数据，无原生代码。
- **新包**：`packages/durable`（持久化会话/任务/文档运行时，Pico v5）：memory/JSONL/SQLite 后端，SQLite 用 Node 内置 `node:sqlite`，无原生绑定；`packages/chord` 依赖面不变（esbuild 仍是唯一原生依赖）→ 均可 `--ignore-scripts` 安装，不新增安装步骤。
- **会话层**：agent-core harness 重构为 Pico5 模型（entry tree + values/lists + Branches/AgentLanes + usage ledger），新增实验性 `pico3/` 内核；coding-agent 自身会话路径不变 → 纯 TS，不影响构建与安装。

> 注：以上含已发布的 v0.87.0/v0.87.1 及至 `5fd446ca1` 的未发布提交，行为可能随后续 release 变化。本轮仅核对代码与文档，未在 QNX 设备上执行冒烟验证。

### 2026-09-26 增量核对（`5fd446ca1` → `d6af72e18`）

本轮合并约 15 个上游提交，未触及 QNX 平台分支、工具下载或剪贴板实现：

- **构建工具链**（ca7460d16）：移除 tsgo native preview 与 tsx，改用 TypeScript 7.0.2（`tsc --noEmit`，ES2024，verbatimModuleSyntax）；示例/测试用 plain `node` + source resolver hook 运行。Node 分发仍为 esbuild 单文件 bundle → `--ignore-scripts` 安装要求与 QNX 部署步骤不变。
- **ai**：openai SDK 升级至 7.19.0 (#10044)；Fast mode service tier 按 priority 计价 (#10034)；Mistral 空 content delta 忽略 (#9674)；模型级 samplingParams 应用于直接 `stream()`/`complete()` 调用 (#9506)——均为纯 TS，无原生代码。
- **会话层**：新会话文件在首个 user/assistant 消息时落盘（此前仅首个 assistant 后），fork 复用同一规则；durable 新增 conversation document fork + typed IDs/ownership——纯 TS。
- **TUI/主题**：自定义主题遵循 truecolor (#9973)；overlay 在 stop() 后关闭时保持光标可见——纯 JS 渲染逻辑，SSH 远程路径不受影响。

> 注：以上为至 `d6af72e18` 的未发布提交（v0.87.1 后的下一版本），行为可能随后续 release 变化。本轮仅核对代码与文档，未在 QNX 设备上执行冒烟验证。
