# ShellAX 项目说明（开发速查）

> 面向后续开发会话：优先读本文件 + 点名源文件，**不要整库通读**。  
> 上游基线：`master` @ `4ed0a8d184db3db88f6bad84393ff88ffa88d407`（v1.3.4，2026-11-05）

---

## 1. 定位 / 仓库 / 版本 / 许可证

**一句话**：Rust + GPUI 的跨平台终端 / SSH 客户端，同窗集成 SFTP、数据库（含达梦）、Redis、Docker、端口转发、小工具，并为 AI Agent 提供权限引擎与审计。

| 项 | 值 |
|---|---|
| 仓库 | https://github.com/daxiangme/ShellAX.git（公开） |
| 默认分支 | `master` |
| Cargo 包名 | `ssh-term-proto`（`publish = false`） |
| 版本 | `1.3.4`（见 `Cargo.toml`） |
| Edition | Rust 2021；工具链 **stable ≥ 1.85** |
| 二进制 | `shellax`（GUI，`src/main.rs`）、`shellax-cli`（`src/bin/shellax-cli.rs`） |
| 许可证 | **AGPL-3.0-only**（`LICENSE`；第三方见 `THIRD_PARTY_NOTICES.md`） |
| 平台 | 发布：Windows x64、macOS aarch64；Linux 可源码构建，无现成发布包 |

**核心栈（勿臆造 API）**：`gpui-pre`/`gpui-pre-platform` **=0.3.9**、`gpui-component`/`gpui-kit-assets` **=0.7.3**、`alacritty_terminal`、`portable-pty`、`russh`/`russh-sftp`、`sqlx`、`redis`、`bollard`、`tiberius`、`mongodb`、`dm-rust`、可选 `duckdb`（`--features duckdb`）、`wasmi`（扩展沙箱）、`keyring` + `chacha20poly1305`（密钥）。

---

## 2. 本地路径与常用命令（Windows 为主）

| 用途 | 路径 |
|---|---|
| 本机克隆 | `C:\Users\king\source\ShellAX` |
| 配置 | `%APPDATA%\ShellAX\config.toml`（可用 `SHELLAX_CONFIG`） |
| 数据（审计/历史/会话/更新） | `%LOCALAPPDATA%\shellax\` |
| GUI 日志（无控制台） | `%LOCALAPPDATA%\shellax\logs\shellax.log`（上次：`shellax.old.log`） |
| 密钥 | Windows 凭据管理器（keyring）；不可用时 `secrets.enc` |
| 扩展包 | `<配置目录>/ext/<包名>/` |
| 组织托管策略 | `/etc/shellax/managed-policy.yaml`（只读、最高优先；Windows 一般无） |

**常用命令**（在仓库根）：

```powershell
cargo check
cargo clippy --all-targets          # 要求 0 警告
cargo test --release                # 需本机 sshd 时见 testenv/；无 sshd 用 --lib
cargo test --release --lib
cargo run                           # 调试 GUI → target\debug\shellax.exe
cargo build --release
cargo build --profile dist --bin shellax --bin shellax-cli   # 产物 target\dist\
pwsh -File tools\package_win.ps1    # 构建+打包 → dist\
```

**环境变量**：`SHELLAX_NO_LOGFILE=1`、`SHELLAX_NO_ANIM=1`、`SHELLAX_SERVER_DECORATIONS=1`、`SHELLAX_UPDATE_URL`、`GPUI_FXC_PATH`（release 缺 fxc 时）。  
**发版三处版本须一致**：`Cargo.toml`、`packaging/windows/installer.nsi`、`packaging/windows/shellax.rc`。

---

## 3. 目录地图

### 根目录

| 路径 | 职责 |
|---|---|
| `src/` | 几乎全部应用代码（GUI 大文件 + lib 模块） |
| `src/bin/shellax-cli.rs` | CLI |
| `docs/` | DEVELOPMENT / PRD / EXTENSIONS / UI_REDESIGN_PLAN / WINDOWS_TEST |
| `agents/*.toml` | **终端内外部 Agent 状态检测规则**（grok/claude/codex 等），**不是** Cursor 助手 brief |
| `extensions/` | WASM 扩展包源码（独立工作区） |
| `vendor/gpui-pre-{windows,macos}/` | GPUI 后端补丁（交叉编译预着色器 / macOS 剪贴板） |
| `packaging/windows/` | NSIS、`shellax.rc` |
| `tools/` | `package_win.ps1` / `.sh`、`e2e_hooks.sh` 等 |
| `testenv/` | 回环 sshd 测试环境 |
| `tests/` | 集成测试 |
| `locales/`、`themes/`、`schemes/`、`assets/` | i18n、主题、配色、资源 |
| `build.rs` | Windows 图标/版本资源 |

### `src/` 关键文件 / 模块（职责表）

| 文件/目录 | 职责 |
|---|---|
| **`main.rs`** | GUI 入口与主状态机（极大，改 UI/模块切换先搜这里） |
| **`lib.rs`** | 库模块清单（无 UI 的核心后端） |
| **`ssh.rs`** | russh 会话、认证、复用、重连 |
| **`local.rs`** | 本机 PTY（portable-pty） |
| **`term.rs` / `termview.rs`** | 终端会话模型 / GPUI 终端视图（termview 极大） |
| **`config.rs`** | 配置读写（大） |
| **`secrets.rs`** | 钥匙串 / 加密文件 |
| **`hosts.rs` / `hosttree.rs` / `host_dialog.rs`** | 主机模型、树、编辑对话框 |
| **`sftp.rs` / `sftp_view.rs` / `dirsync.rs` / `clipfiles.rs`** | SFTP、双栏 UI、目录同步、剪贴板粘贴文件路径 |
| **`db/` + `db_*.rs`** | 多引擎驱动、树、编辑器、视图 |
| **`doc/` + `doc_view.rs`** | MongoDB 等文档库 |
| **`kv/` + `kv_*.rs`** | Redis |
| **`dock/` + `dock_view.rs`** | Docker（bollard） |
| **`forward.rs` / `port_view.rs` / `proxy.rs`** | 端口转发与代理 |
| **`tools/` + `tools_*.rs` / `ext/` + `ext_ui.rs`** | 内置小工具 + WASM 扩展运行时 |
| **`policy.rs` / `policy_server.rs` / `managed.rs` / `audit.rs`** | 权限引擎、本地策略服务、托管策略、审计 JSONL |
| **`ai.rs` / `ai_ctx.rs` / `ai_mention.rs` / `chat_view.rs`** | 内置 AI、上下文、# 提及、聊天 UI |
| **`agent.rs` / `hooks.rs`** | 外部 Agent 状态识别、命令钩子 |
| **`db_mcp/` + `mcp_*.rs`** | 本机数据库 MCP（MySQL/达梦等） |
| **`updater.rs` / `update.rs` / `update_ui.rs`** | 检查/下载/校验/退出安装 |
| **`sidebar.rs` / `panels.rs` / `workspace.rs` / `settings_view.rs`** | 侧栏、面板、工作区、设置页 |
| **`brew*.rs`** | macOS Homebrew 模块 |
| **`daemon.rs`** | Unix 会话守护（关窗保 shell） |
| **`platform.rs` / `chrome.rs` / `theme.rs` / `i18n.rs`** | 平台差异、窗口装饰、主题、国际化 |

> 命名习惯：`*_view.rs` / `*_ui.rs` / `*_dialog.rs` 多为 GPUI 层；无后缀或 `mod.rs` 多为协议/业务后端。

---

## 4. 功能模块 ↔ 代码大致对应

| 产品模块 | 优先阅读 |
|---|---|
| 主机 / 本地终端 / SSH | `hosts*`、`hosttree`、`ssh`、`local`、`term`、`termview`、`suggest`、`session_store`、`layout`、`panels` |
| SFTP / 目录同步 / 粘贴路径 | `sftp`、`sftp_view`、`sftp_follow`、`dirsync`、`filelist`、`clipfiles` |
| 数据库（SQL） | `db/*`、`db_view`、`db_tree`、`db_editor`、`db_dialog`、`db_ui` |
| MongoDB 等文档 | `doc/`、`doc_view` |
| Redis | `kv/`、`kv_view`、`kv_tree`、`kv_ui` |
| Docker | `dock/`、`dock_view` |
| 端口转发 | `forward`、`port_view`、`proxy` |
| 工具 + WASM 扩展 | `tools/`、`tools_view`、`ext/`、`ext_ui`；包格式见 `docs/EXTENSIONS.md` |
| Homebrew（仅 macOS） | `brew`、`brew_ui`、`brew_view` |
| AI 助手 | `ai`、`ai_ctx`、`ai_mention`、`chat_view` |
| 权限 / 审计 / 外部 Agent | `policy`、`policy_server`、`managed`、`audit`、`agent`、`hooks`、`bin/shellax-cli`；规则数据 `agents/*.toml` |
| 数据库 MCP | `db_mcp/`、`mcp_backend`、`mcp_view` |
| 主机工具卡（进程/服务等） | `hosttools`、`host_tools_view`、`hoststats*` |
| 自动更新 | `updater`、`update`、`update_ui` |
| 设置 / 主题 / 右键菜单 | `settings_view`、`config`、`theme`、`ctxmenu*` |

---

## 5. 给助手的硬约束

1. **密钥**：密码、API Key **只**进钥匙串 / `secrets.enc`；**禁止**写入 `config.toml`、日志、审计正文。  
2. **Clippy**：提交前 `cargo clippy --all-targets` **0 警告**。  
3. **阅读策略**：先本 brief → 点名文件；需要环境/打包再读 `docs/DEVELOPMENT.md`；**禁止**每次通读 README/PRD/全 `src/`。  
4. **上游推送**：当前连接账号对该仓 **pull:true, push:false** → **只改本地**，勿 `git push`、勿开上游 PR。  
5. **`agents/`**：是运行时 Agent 状态规则，不是本项目的 Cursor 开发说明。  
6. **发版**：改版本号必须同步 `Cargo.toml` + NSIS + `shellax.rc`；安装包文件名与 `updater` 耦合，勿随意改名。  
7. **可选特性**：DuckDB 需 `--features duckdb`，发布包默认不含。

---

## 6. 深读索引（何时读哪份）

| 文档 | 何时打开 |
|---|---|
| **本文件** | 每次开新开发会话的默认入口 |
| `docs/DEVELOPMENT.md` | 环境依赖、日志路径、打包、交叉编译、macOS 首次构建、更新安装流程 |
| `README.md` / `README.en.md` | 功能概览、安装说明（勿整篇粘贴进上下文） |
| `docs/EXTENSIONS.md` | 做/改 WASM 扩展包、清单与沙箱约定 |
| `docs/UI_REDESIGN_PLAN.md` | 改视觉/布局规则、间距与组件约定 |
| `docs/WINDOWS_TEST.md` | Windows 手工/回归测试清单 |
| `docs/PRD.md` | **巨大**；仅在需要产品需求细节时按章节检索，勿全文加载 |
| `THIRD_PARTY_NOTICES.md` / `licenses/` | 许可证合规 |

---

## 7. 当前已知限制

- **无上游写权限**：开发成果留在 `C:\Users\king\source\ShellAX`（或自有 fork）；不要尝试推 `daxiangme/ShellAX`。  
- **macOS**：代码已写（原生按钮、毛玻璃、Metal），但文档明确 **尚未在真机编译运行**；首次构建可能要修小问题。  
- **Linux**：可构建；系统托盘等有 Linux 专用代码；**无官方发布包**；剪贴板粘贴文件路径 **Linux 未实现**（`clipfiles`）。  
- **签名**：Windows 安装包未代码签名（SmartScreen）；macOS ad-hoc、未公证。  
- **数据库 MCP**：README 称目前侧重 **MySQL 与达梦**；其它引擎以 UI/驱动为准，勿假设已有 MCP。  
- **`main.rs` / `termview.rs` / `settings_view.rs` / `sftp_view.rs` 等极大**：改功能用符号搜索 + 局部阅读，避免整文件塞进上下文。

---

## 速查：CLI 子命令（`shellax-cli`）

`status` · `check` · `hook`/`install` · `serve` · `hosts`/`exec`/`upload`/`download` · `play` · `daemon`（仅 Linux/Mac）

---

*生成说明：结构来自上游 `master` 目录与 `docs/DEVELOPMENT.md` / `README.md` / `Cargo.toml` / `src/lib.rs`；未臆造公开 API 细节。*
