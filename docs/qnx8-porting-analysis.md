# Pi → QNX 8.0 移植可行性分析

> 分析日期：2026-08（基于 GitHub 公开仓库证据）
> 范围：将 pi monorepo（`packages/coding-agent` 主 CLI/SDK）移植到 QNX Neutrino 8.0（x86_64 与 aarch64le 均适用）

## 结论

**可行**。核心阻塞（Node.js 运行时）已被 QNX 官方解决：QNX 8.0 通过 apk 提供 Node.js 24.14.1 LTS。主要剩余工作量为 ripgrep/fd 两个外部工具的 QNX 构建，以及实际冒烟验证。

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
| npm 安装 | ✅ | pi 依赖树全纯 JS + WASM：`@silvia-odwyer/photon-node` 是 WASM（`photon_rs_bg.wasm`）；`@mariozechner/clipboard` 是 optionalDependency（napi 平台预编译，QNX 无则优雅降级）；无 node-gyp 编译 |
| bash 工具 | ✅ | aports 有 bash 5.3；`shell.ts` 路径 `/bin/bash` → PATH bash → `sh` 兜底成立 |
| git | ✅ | aports 有 git；footer 分支显示可用 |
| grep 工具 | ❌ 硬缺口 | 需要 `rg`；pi 的自动下载（`tools-manager.ts`）只支持 darwin/linux/win32 资产 |
| find 工具 | ⚠️ 缺口 | 需要 `fd`（有 `systemBinaryNames` 可先用系统命令） |
| TUI | ⚠️ 受限 | kitty keyboard protocol、bracketed paste、终端图片依赖连接的终端模拟器（SSH 登录场景可用）；`native-modifiers.ts` 无 QNX 预编译 → `isNativeModifierPressed` 恒 false，降级不崩溃 |
| 剪贴板 | ⚠️ 降级 | `clipboard-native.ts` 依赖 `@mariozechner/clipboard`，QNX 无预编译 → 返回 null |
| headless 模式 | ✅ | `--mode rpc / print / json` 不依赖 TUI |
| OAuth | ⚠️ 需验证 | 无浏览器环境需走 device code 流程；Copilot/Radius/Kimi 已有实现（`packages/ai/src/auth/oauth/device-code.ts`，github-copilot 轮询带 429 重试），其余 provider 逐个确认 |

## 剩余缺口与解决路径

### ripgrep / fd（唯一硬缺口）

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

注意：若 `process.platform === "qnx"`，`getAssetName()` 对未知平台返回 `null` → 自动下载被跳过，不会误下载 linux 二进制。

## 落地步骤

1. QNX 8.0 配置 apk 源，`sudo apk add nodejs npm bash git`
2. 准备 rg/fd（交叉编译或 aports PR）
3. `npm install -g @earendil-works/pi-coding-agent`（纯 JS 包）→ `pi --version` 冒烟
4. 先跑 headless（`pi -p "..."`），再在 SSH 终端验证交互模式
5. 处理 OAuth（无浏览器时验证 device code 流程）
6. 可选用 Node 分发版（`dist/cli.js`）而非 Bun 二进制

## 参考链接

- https://github.com/qnx-ports/aports （APKBUILD: extra/nodejs, core/bash, core/git）
- https://github.com/qnx-ports/aports/pull/354 （nodejs aport）
- https://github.com/qnx-ports/node （QNX Node fork，tag qnx-v24.14.1）
- https://github.com/qnx/claude-code-qnx （Claude Code on QNX 官方示例）
- https://github.com/qnx-ports/aports/pull/398 （llama.cpp aport）

## 备注

- 初次分析（未搜索 GitHub 前）曾判断"Node 需自行移植、整体不建议"，该结论已被官方 aports 证据推翻，本文档为修正版。
- Bun 二进制分发版不适用 QNX（Bun 未移植），使用 Node 分发。
