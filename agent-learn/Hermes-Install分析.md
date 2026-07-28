[TOC]


# Windows安装

Windows下 Hermes 的安装使用的是 `scripts/install.ps1`，即使使用exe安装文件，背后也是下载执行的此脚本。
我不是很喜欢Hermes的这种安装方式，它会创建一个 `~/AppData/Local/hermes` 文件夹，在其中安装一个单独的UV并在`Path`中设置路径环境变量，屏蔽我自己安装的UV。

对于 MacOS/Linux 环境开发者来说，可以使用 `setup-hermes.sh` 来初始化开发环境，但对于Windows环境似乎没有对应脚本。

以下是基于 Hermes 源码搭建 Windows 开发/运行环境的总结。

-----------------------------------------------------------------------
## `install.ps1`安装分析

`scripts\install.ps1` 是 Hermes 在 Windows 平台的官方安装脚本（约 3400 行），设计为**完全自包含、用户级安装**，
所有组件**默认**安装到 `%LOCALAPPDATA%\hermes\` 下，不需要管理员权限。

### 设置的环境变量

| 变量 | 作用域 | 用途 |
|---|---|---|
| `HERMES_HOME` | User (持久) + Session | 核心变量，默认 `%LOCALAPPDATA%\hermes`，所有配置/数据/日志的根目录 |
| `HERMES_GIT_BASH_PATH` | User (持久) + Session | 指向 `bash.exe` 的绝对路径，供 Hermes 找到 Git Bash |
| `UV_INSTALL_DIR` | Session (临时) | 安装 uv 时设置，指向 `$HermesHome\bin` |
| `UV_PYTHON` | Session (临时) | 固定到 venv 的 Python 解释器，防止用户环境的 `UV_PYTHON` 覆盖 |
| `VIRTUAL_ENV` | Session (临时) | 指向 venv 目录 |
| `UV_PROJECT_ENVIRONMENT` | Session (临时) | 防止现代 uv (≥0.5) 将 `uv sync` 安装到 sibling `.venv\` 而非 `$InstallDir\venv\`。uv 0.5+ 默认将依赖安装到项目根目录下的 `.venv\`（而非 `venv\`），设置此变量强制指向正确的 venv 路径 |
| `AGENT_BROWSER_EXECUTABLE_PATH` | 写入 `$HermesHome\.env` | 用户显式指定浏览器路径时写入 |
| `GIT_CONFIG_COUNT/KEY_0/VALUE_0` | Session (临时) | 设置 `windows.appendAtomically=false` 解决 Windows git 原子写入问题 |
| `CSC_IDENTITY_AUTO_DISCOVERY` 等 | Session (临时) | 仅在 `-IncludeDesktop` 构建时清除签名相关变量 |
| `ELECTRON_MIRROR` | Session (临时) | Electron 下载失败时临时设为国内镜像 |
| `TEMP` / `TMP` | Session (临时) | 将 8.3 短路径名展开为长路径 |
| `ProgressPreference` | Session (临时) | 设为 `SilentlyContinue`，抑制 `Invoke-WebRequest` 进度条避免下载速度骤降 |
| `Path` (User) | 持久 | 追加 `$HermesHome\git\cmd`、`$HermesHome\git\bin`、`$HermesHome\git\usr\bin`（由 `Install-Git` 添加）、`$HermesHome\node`（由 `Test-Node` 便携版安装时添加）、`$InstallDir\venv\Scripts`（由 `Set-PathVariable` 添加） |

> **注意**：`$HermesHome\bin`（uv 安装目录）**不会**被添加到 User PATH。`Set-PathVariable` 只添加 `$InstallDir\venv\Scripts`，Git 和 Node 各自在其安装函数内部独立管理 PATH 条目。

其中 `%LOCALAPPDATA%`、`$HermesHome`、`$InstallDir` 这3个环境变量的默认值和它们之间的关系如下：

- 默认值（未设置 `HERMES_HOME` 环境变量时）

| 变量             | 默认值                               | 含义                                           |
| ---------------- | ------------------------------------ | ---------------------------------------------- |
| `%LOCALAPPDATA%` | `C:\Users\<name>\AppData\Local`      | Windows 系统环境变量，用户级应用数据根目录     |
| `$HermesHome`    | `%LOCALAPPDATA%\hermes`              | Hermes **所有数据**的根目录                    |
| `$InstallDir`    | `%LOCALAPPDATA%\hermes\hermes-agent` | 源码 clone 位置，**嵌套在 `$HermesHome` 内部** |

这里和官方文档的描述略有出入，官方文档基于跨平台的考虑，采用的统一描述是用户数据目录为`~/.hermes`，但是根据`install.ps1`里的默认设置，实际上用户数据是在`%LOCALAPPDATA%\hermes`里。

实际上，官方文档是将 `~/.hermes` 作为**跨平台的泛称**——在 Unix 上它就是字面意思，在 Windows 上下文中它是对 `HERMES_HOME` 的简写。

- 层级关系

```
%LOCALAPPDATA%                          ← Windows 系统路径
└── hermes\                             ← $HermesHome（配置/数据根目录）
    ├── .env                            ← API keys
    ├── config.yaml                     ← 行为配置
    ├── logs\                           ← agent.log, errors.log
    ├── sessions\                       ← SQLite session DB
    ├── skills\                         ← 用户 skills
    ├── bin\uv.exe                      ← 独立安装的 uv
    ├── git\                            ← PortableGit
    ├── node\                           ← 便携 Node.js
    └── hermes-agent\                   ← $InstallDir（源码仓库）
        ├── .git\
        ├── venv\                       ← Python 虚拟环境
        ├── run_agent.py
        └── ...
```

- 如果已设置 `HERMES_HOME` 环境变量，两个变量都会基于 `HERMES_HOME` 重新推导：

| 变量          | 值                                                        |
| ------------- | --------------------------------------------------------- |
| `$HermesHome` | `$env:HERMES_HOME`（直接使用，不再拼接 `%LOCALAPPDATA%`） |
| `$InstallDir` | `$env:HERMES_HOME\hermes-agent`                           |


### 下载/安装的组件

| 阶段 | 组件 | 来源 | 安装位置 |
|---|---|---|---|
| uv | Astral `uv` 包管理器 | `https://astral.sh/uv/install.ps1` | `$HermesHome\bin\uv.exe` |
| Python | CPython 3.11 (fallback: 3.12→3.13→3.10) | uv 自动下载 (PyPI) | uv 管理的 Python 目录 |
| Git | PortableGit (含 bash.exe) | GitHub Releases (v2.54.0) | `$HermesHome\git\` |
| Node.js | Node.js 22 LTS 便携版 | `https://nodejs.org/dist/` | `$HermesHome\node\` |
| ripgrep | `BurntSushi.ripgrep.MSVC` | winget / choco / scoop | 系统级 |
| ffmpeg | `Gyan.FFmpeg` | winget / choco / scoop | 系统级 |
| Repository | `hermes-agent` 源码 | GitHub (SSH→HTTPS→ZIP fallback) | `$InstallDir` (默认 `$HermesHome\hermes-agent`) |
| Python venv | 虚拟环境 | uv 创建 | `$InstallDir\venv\` |
| Python 依赖 | hermes-agent[all] + 传递依赖 | PyPI (通过 `uv sync --locked`) | `$InstallDir\venv\` |
| Node 依赖 | npm workspace 依赖 | npm registry | `$InstallDir\node_modules\` |
| Playwright Chromium | 浏览器引擎 | Playwright CDN | `%LOCALAPPDATA%\ms-playwright\` |
| agent-browser | `agent-browser@^0.26.0` + `@askjo/camofox-browser@^1.5.2` | npm 全局安装 | `$HermesHome\node\` |
| Desktop (可选) | Electron + Hermes.exe | npm + GitHub Electron releases | `$InstallDir\apps\desktop\release\win-unpacked\` |
| Skills | 内置 skills | 本地复制 | `$HermesHome\skills\` |
| 配置模板 | `.env`、`config.yaml`、`SOUL.md` | 从模板复制 | `$HermesHome\` |
| Bootstrap 标记 | `.hermes-bootstrap-complete` | 写入 JSON | `$InstallDir\` |
| Desktop 快捷方式(可选) | Start Menu + Desktop `.lnk` | 通过 WScript.Shell COM 创建 | `%Programs%\Hermes.lnk`、`Desktop\Hermes.lnk` |


关于UV，脚本在 `Install-Uv` 函数中的策略是：

```powershell
$managedUv = Join-Path $HermesHome "bin\uv.exe"   # 例如 C:\Users\data-\AppData\Local\hermes\bin\uv.exe
```

**它安装了一个"受管理的 uv"到 `$HermesHome\bin\uv.exe`**，逻辑如下：

1. 先检查 `$HermesHome\bin\uv.exe` 是否已存在 → 若存在，直接使用，不重新下载
2. 若不存在，设置 `$env:UV_INSTALL_DIR = "$HermesHome\bin"`，然后运行 astral 官方安装脚本 `irm https://astral.sh/uv/install.ps1 | iex`
3. 若官方安装失败，提示用户手动安装

**是否会覆盖已有的 UV？**

- ✅ 不会覆盖系统级 uv：它把 uv 安装到 `%LOCALAPPDATA%\hermes\bin\uv.exe`，不碰 `%USERPROFILE%\.local\bin\uv.exe` 或 `%USERPROFILE%\.cargo\bin\uv.exe`。
- ✅ 不会修改已有的 PATH 中的 uv：脚本将 `$HermesHome\bin` 加到 User PATH 的**最前面**（`Set-PathVariable`），但这只影响 `hermes.exe` 的查找。uv 本身仅在脚本内部通过 `$script:UvCmd = $managedUv` 绝对路径调用，不会通过 PATH 查找。
- ⚠️ 潜在 PATH 冲突：`$HermesHome\bin` 被加到 User PATH 最前面，如果自己也有 uv 在 PATH 中，新终端里 `uv` 命令可能优先找到 Hermes 管理的版本。但这只影响终端中直接敲 `uv` 的情况，且 Hermes 内部始终用绝对路径调用自己的 uv。

注意：`Set-PathVariable` 函数负责将 `$HermesHome\bin` 和 `venv\Scripts` 写入 User PATH 最前面，这意味着 Hermes 管理的 uv 和 hermes.exe 在 PATH 中优先级最高。

**如果不想让 Hermes 安装独立的 uv**，可以：

- 提前创建 `$HermesHome\bin\uv.exe`（或 copy 你自己的 uv 过去），`Install-Uv` 检测到已存在会跳过安装
- 但脚本中后续所有 uv 调用都硬编码为 `$script:UvCmd`（即该路径），无法绕过


### 执行入口

安装完成后，`$InstallDir\venv\Scripts\hermes.exe` 被添加到 User PATH，新终端中可直接使用：

```powershell
hermes              # 启动交互式聊天 (CLI)
hermes setup        # 配置 API key
hermes gateway      # 启动消息网关 (Telegram/Discord 等)
hermes update       # 更新到最新版本
```

核心启动链：`hermes.exe` → `venv\Scripts\python.exe` → `hermes_cli.main` 模块。

### 完整的阶段执行顺序

```
uv → python → git → node → system-packages → repository →
venv → dependencies → node-deps → [desktop(可选)] →
path → config-templates → platform-sdks → bootstrap-marker →
configure(交互) → gateway(交互)
```

### 关键设计特点

- **自包含**：所有组件安装到 `%LOCALAPPDATA%\hermes\`（`HermesHome`默认值），不依赖系统已有工具
- **用户级**：不需要管理员权限，不修改系统级配置（除可选的 winget 安装 ripgrep/ffmpeg）
- **独立 UV**：安装自己的 uv 到 `$HermesHome\bin\uv.exe`，并将该目录加入 User PATH 最前面，可能覆盖用户自己安装的 UV
- **独立 Node.js**：下载便携版 Node.js，不依赖系统 Node.js
- **独立 Git**：下载 PortableGit，不依赖系统 Git


### 补充说明

（1）**Python 依赖安装的分级回退策略**

`Install-Dependencies` 采用四级回退策略，确保部分依赖不可用时仍能完成安装：

| 层级 | 策略 | 说明 |
|---|---|---|
| Tier 0 | `uv sync --extra all --locked` | 哈希验证安装（优先），通过 `uv.lock` 中的 SHA256 校验所有传递依赖 |
| Tier 1 | `uv pip install -e .[all]` | 从 PyPI 解析所有 curated extras |
| Tier 2 | `uv pip install -e .[safeAll]` | 去除 `$brokenExtras` 列表中的已知问题 extras |
| Tier 3 | `uv pip install -e .` | 最简安装，仅核心 CLI，无任何 extras |

之后还有**基线导入验证**（检查 `dotenv/openai/rich/prompt_toolkit` 可导入）和 **Dashboard 依赖验证**（检查 `fastapi/uvicorn`）。

（2）**环境变量补充说明**

脚本中还有两个 Session 临时变量值得一提：

- **`ProgressPreference`**：设为 `SilentlyContinue`，抑制 PowerShell 的 `Invoke-WebRequest` 逐字节进度条。Windows PowerShell 5.1 的进度 UI 对每个收到的字节同步重绘，在下载大文件（如 57MB 的 PortableGit）时会将下载速度降低 10-100 倍。
- **`[Console]::OutputEncoding`**：强制设为 UTF-8，确保 npm/playwright 等原生命令的 box-drawing 字符和 Unicode 输出正确渲染，而非被 IBM437/Windows-1252 错误解码。

（3）**配置模板阶段额外创建的文件**

`Copy-ConfigTemplates` 除了 `.env` 和 `config.yaml`，还会创建 **`SOUL.md`**，这是一个全局 persona 文件，内容与 `hermes_cli/default_soul.py` 中的 `DEFAULT_SOUL_MD` 保持一致。

此外，该阶段还会创建 `$HermesHome` 下的子目录结构（`cron`、`sessions`、`logs`、`pairing`、`hooks`、`image_cache`、`audio_cache`、`memories`、`skills`），并调用 `tools\skills_sync.py` 将内置 skills 同步到 `$HermesHome\skills`。

**关键实现细节**：使用 `.NET` 的 `UTF8Encoding($false)` 直接写入（无 BOM），因为 PowerShell 5.1 的 `Set-Content -Encoding UTF8` 默认带 BOM，而 Hermes 的 prompt-injection 扫描器会将 BOM 标记为不可见 Unicode 字符并拒绝加载。

（4）**Bootstrap 标记文件**

`Write-BootstrapMarker` 在 `$InstallDir\.hermes-bootstrap-complete` 写入一个 JSON 文件，告知 Desktop 应用 "install.ps1 已成功运行，无需触发传统的首次启动 bootstrap"。
结构包含 `schemaVersion`、`pinnedCommit`、`pinnedBranch`、`completedAt`。同样使用 BOM-less UTF-8 写入，因为 Node.js 的 `JSON.parse` 拒绝 BOM。

（5）**Platform SDK 验证阶段**

`Install-PlatformSdks` 阶段扫描 `.env` 中已配置的 Messaging Token，按需验证和补救对应的 SDK：

| Token 环境变量 | 验证的导入 | 补救安装的 pip 包 |
|---|---|---|
| `TELEGRAM_BOT_TOKEN` | `telegram` | `python-telegram-bot[webhooks]>=22.6,<23` |
| `DISCORD_BOT_TOKEN` | `discord` | `discord.py[voice]>=2.7.1,<3` |
| `SLACK_BOT_TOKEN` | `slack_sdk` | `slack-sdk>=3.27.0,<4` |
| `SLACK_APP_TOKEN` | `slack_bolt` | `slack-bolt>=1.18.0,<2` |
| `WHATSAPP_ENABLED` | `qrcode` | `qrcode>=7.0,<8` |

由于 `uv` 创建的 venv 不含 pip，该阶段会先通过 `python -m ensurepip --upgrade` 引导安装 pip，再逐个安装缺失的 SDK。

（6）**Desktop 构建的容错机制**

`Install-Desktop`（仅 `-IncludeDesktop` 时触发）包含多层容错：

1. **npm 安装**：先尝试 `npm ci`（从 lockfile 精确安装），失败后回退到 `npm install`
2. **Electron dist 自愈**：若 npm 安装失败且检测到 Electron 的 `dist/` 目录缺失，自动调用 `Try-RestoreElectronDist` 修复
3. **构建重试**：`npm run pack` 失败后自动清除 Electron 下载缓存（`Clear-ElectronBuildCache`）并重试一次
4. **镜像回退**：若 GitHub 下载仍失败，自动切换到 `https://npmmirror.com/mirrors/electron/` 国内镜像重试
5. **构建后**：自动创建 Start Menu 和 Desktop 快捷方式，并通过 `ie4uinit.exe -show` 刷新图标缓存


-----------------------------------------------------------------------
## 目标

基于已 clone 的 Hermes 源码，使用系统已有的 UV、Git、Node.js，在**不运行 `install.ps1`** 的前提下，手动搭建支持 CLI / TUI / Desktop 三种启动方式的开发环境。


-----------------------------------------------------------------------
## 前提条件

| 组件     | 状态                                         |
| ------- | -------------------------------------------- |
| Git     | ✅ 已安装                                    |
| UV      | ✅ 已安装                                    |
| Node.js | ✅ v22.22，满足 `>=22.12` 的 Desktop 构建要求 |
| 仓库    | ✅ 源码已 clone 到本地（当前目录）             |


关键原理：

（1）Hermes 通过 `HERMES_HOME` 环境变量决定配置/数据目录。
- 默认值是 `%LOCALAPPDATA%\hermes`，只需设置该变量即可重定向到任意位置。
- Python 代码中所有路径查找都通过 `get_hermes_home()` 读取此变量（见 `hermes_constants.py`），不硬编码路径。

（2）TUI 和 Desktop 对 Node.js 的依赖：
- TUI: `_launch_tui()` 在 `main.py` 中执行，它 spawn 一个 Node.js 子进程运行 ui-tui 下的 Ink 应用。生产模式读取 `dist/entry.js`（esbuild 打包产物），`--dev` 模式使用 `tsx src/entry.tsx`（热重载）。
- Desktop: Electron 主进程（`main.cjs`）启动后 `spawn hermes serve` 子进程作为后端。开发模式下 Vite + Electron 各自独立进程，前端热重载。



-----------------------------------------------------------------------
## HERMES_HOME 路径说明

Hermes 通过环境变量 `HERMES_HOME` 决定配置/数据/日志目录。

核心逻辑在 `hermes_constants.py::_get_platform_default_hermes_home()`：

```shell
# Windows: %LOCALAPPDATA%\hermes  (即 C:\Users\<name>\AppData\Local\hermes)
# Linux/macOS: ~/.hermes
```

安装脚本 `install.ps1` 默认将 `HERMES_HOME` 设置为 `%LOCALAPPDATA%\hermes` 并写入 User 环境变量，并向其中安装独立 UV、Node、Git 等。

**这里会跳过这些内容**，将 `HERMES_HOME` 设为当前项目内的 `.hermes` 目录，完全不影响系统环境。

-----------------------------------------------------------------------
## 执行步骤

以下所有命令在已经clone的仓库根目录 `D:\Path\to\hermes-agent` 下执行。

### 1. 安装 Python 依赖

使用 UV 工具安装依赖：

```powershell
# 安装运行依赖（含 dev 开发依赖，debugpy、pytest、ruff 等）
uv sync --extra dev
```

`uv sync` 会自动创建 `.venv` 虚拟环境并安装 `pyproject.toml` 中 `[project] dependencies` 声明的核心依赖。
`--extra dev` 额外安装 `[project.optional-dependencies] dev` 中的开发工具：`debugpy`、`pytest`、`pytest-asyncio`、`ruff`、`mcp`、`setuptools` 等。

> **注意**：`uv sync` 默认只安装核心依赖，不安装 `[all]` extra。`[all]` 中包含 `messaging`（Telegram/Discord/Slack 等平台 SDK）、`matrix`（mautrix 加密）等大量可选组件。
> 这些可选组件在 Hermes 中通过 `tools/lazy_deps.py` 按需懒加载——当用户首次使用某个 provider 或平台时自动安装，无需手动预装。

### 2. 安装 Node.js workspace 依赖

```powershell
npm ci
```

此命令安装所有 workspace：`ui-tui`（TUI）、`apps/desktop`（Electron Desktop）、`web`（Dashboard）。
首次需下载 Electron 二进制（~150MB），耗时较长。

**如果 Electron 下载失败**（国内网络），先设置镜像：

```powershell
$env:ELECTRON_MIRROR = "https://npmmirror.com/mirrors/electron/"
npm ci
```

### 3. 创建 HERMES_HOME 目录结构

```powershell
# 假设要放到当前目录下
$hermesHome = ".hermes"

New-Item -ItemType Directory -Force -Path "$hermesHome\logs"
New-Item -ItemType Directory -Force -Path "$hermesHome\sessions"
New-Item -ItemType Directory -Force -Path "$hermesHome\skills"
New-Item -ItemType Directory -Force -Path "$hermesHome\cron"
New-Item -ItemType Directory -Force -Path "$hermesHome\memories"
New-Item -ItemType Directory -Force -Path "$hermesHome\pairing"
New-Item -ItemType Directory -Force -Path "$hermesHome\hooks"
New-Item -ItemType Directory -Force -Path "$hermesHome\image_cache"
New-Item -ItemType Directory -Force -Path "$hermesHome\audio_cache"
```

### 4. 复制配置文件

```powershell
# .env — API keys
Copy-Item ".env.example" "$hermesHome\.env"
# config.yaml — 行为配置
Copy-Item "cli-config.yaml.example" "$hermesHome\config.yaml"
```

编辑 `.hermes\.env`，至少填入一个 API Key（如 `OPENAI_API_KEY=sk-xxx`）。

> **关于 SOUL.md**：`install.ps1` 的 `Copy-ConfigTemplates` 阶段还会从 `hermes_cli/default_soul.py` 中的 `DEFAULT_SOUL_MD` 内容创建 `SOUL.md`（全局 persona 文件）。
> 开发环境中如果不创建此文件，Hermes 会使用内置默认 persona，不影响正常运行。如需自定义，可手动创建 `.hermes\SOUL.md`：
>
> ```powershell
> # 可选：创建 SOUL.md 全局 persona 文件（与 install.ps1 行为一致）
> # 以下内容可以从 install.ps1 中进行复制，但是要注意编码。
> @"
> You are Hermes Agent, an intelligent AI assistant created by Nous Research. You are helpful, knowledgeable, and direct. You assist users with a wide range of tasks including answering questions, writing and editing code, analyzing information, creative work, and executing actions via your tools. You communicate clearly, admit uncertainty when appropriate, and prioritize being genuinely useful over being verbose unless otherwise directed below. Be targeted and efficient in your exploration and investigations.
> "@ | Out-File -FilePath "$hermesHome\SOUL.md" -Encoding utf8NoBOM
> ```

### 5. 设置 HERMES_HOME 环境变量

```powershell
$hermesHome = (Resolve-Path ".hermes").ProviderPath
# 当前终端会话
$env:HERMES_HOME = $hermesHome
# 持久化（可选，省去每次手动设置）
[Environment]::SetEnvironmentVariable("HERMES_HOME", $hermesHome, "User")
```

注意，`SetEnvironmentVariable` 会持久化到注册表（User 级别），新终端自动生效。

### 6. 同步 Skills（可选）

```powershell
.venv\Scripts\python.exe tools\skills_sync.py
```

-----------------------------------------------------------------------
## 三种模式说明

配置好后，Hermes 主要提供了三种用户交互界面：

| 入口           | 类型         | 技术栈                           |
| -------------- | ------------ | -------------------------------- |
| **CLI** (默认) | 终端文本界面 | Python `prompt_toolkit` + `rich` |
| **TUI**        | 终端图形界面 | Node.js Ink (React) 渲染在终端   |
| **Desktop**    | 独立窗口应用 | Electron + React                 |

---
### CLI 模式

```powershell
.venv\Scripts\python.exe -m hermes_cli.main
```

纯文本交互式对话，基于 `prompt_toolkit` + `rich`，**不依赖 Node.js**，最轻量的启动方式。

---
### TUI 模式

```powershell
# 入口同 CLI 模式，使用 --tui 参数区分
.venv\Scripts\python.exe -m hermes_cli.main --tui
# TUI 热重载开发模式
.venv\Scripts\python.exe -m hermes_cli.main --tui --dev
```

`--dev` 使 TUI 使用 `tsx src/entry.tsx`（源文件直接运行），修改 `ui-tui/src/` 后重启即生效。
TUI 的 Ink 渲染器通过 JSON-RPC stdio 与 Python 后端 `tui_gateway` 通信。

> **原理**：`cmd_chat()` 检测到 `--tui` 后调用 `_launch_tui()` → `_make_tui_argv()`。
> `--dev` 模式下，`_make_tui_argv()` 构建 `tsx src/entry.tsx` 命令（而非 `node dist/entry.js`），
> 同时自动执行 `npm install`（如 `node_modules` 缺失）和 `@hermes/ink` 包的预构建。
> Python 进程通过 `subprocess.call()` spawn Node 子进程，自身等待子进程退出。

---
### Desktop 模式

（1）开发模式：直接启动Vite + Electron

```powershell
# dev 表示启动开发模式（Vite 热重载 + Electron 窗口）
npm run dev --workspace apps/desktop
```

这会同时启动 Vite dev server（`localhost:5174`，前端 HMR）和 Electron 窗口（自动打开）。
修改 `apps/desktop/src/` 下的 React 代码会即时反映；Electron 主进程修改需 Ctrl+C 重启后重新运行。

（2）生产模式：构建Electron应用的exe后启动

```powershell
# 一条命令搞定：安装依赖 → 构建 → 启动 Electron
npx hermes desktop
# 或者使用 CLI/TUI 同样的入口，使用 desktop 子命令做区分
.venv\Scripts\python.exe -m hermes_cli.main desktop
```

这个命令内部做的事（对应 `cmd_gui` 函数）：
1. 检查是否有内容哈希构建标记（`$HERMES_HOME/desktop-build-stamp.json`）——若源码未变更则**跳过构建**，直接启动；
2. 若需要构建：`npm ci`（根目录 workspace 依赖，优先；失败后回退到 `npm install`）；
3. `npm run pack`（在 desktop 下：tsc 编译 → vite 打包 → electron-builder 产出 win-unpacked/Hermes.exe），写入新的构建标记；
4. 以子进程启动打包好的Electron应用 `Hermes.exe`。

> **内容哈希跳过机制**：`cmd_gui` 使用 `_desktop_build_needed()` 对比源码树的 SHA-256 哈希与上次成功构建的标记。如果源码未变更，即使多次执行 `hermes desktop` 也不会重复构建，大幅加速重复启动。使用 `--force-build` 可强制重新构建。

支持启动参数设置：
```powershell
# 仅构建不启动（--build-only）
.venv\Scripts\python.exe -m hermes_cli.main desktop --build-only
# 跳过构建直接启动已有打包产物（--skip-build）
.venv\Scripts\python.exe -m hermes_cli.main desktop --skip-build
# 源码模式：electron . 直接跑 dist/（不打包成 exe）
.venv\Scripts\python.exe -m hermes_cli.main desktop --source
```

### hermes 子命令与 .exe 入口

`pyproject.toml` 的 `[project.scripts]` 声明了三个可执行入口：

```toml
[project.scripts]
hermes       = "hermes_cli.main:main"
hermes-agent = "run_agent:main"
hermes-acp   = "acp_adapter.entry:main"
```

`uv sync` 后，setuptools 在 `.venv\Scripts\` 下自动生成对应的 `.exe` wrapper文件：

| 文件 | 入口 | 用途 |
|---|---|---|
| `hermes.exe` | `hermes_cli.main:main` | 交互式 CLI 主程序，包括 CLI 和 TUI，通过有无参数 --tui 区分，也支持 Desktop 模式启动 |
| `hermes-agent.exe` | `run_agent:main` | 单次 agent 调用 (`AIAgent`)，无交互界面 |
| `hermes-acp.exe` | `acp_adapter.entry:main` | ACP 适配器，VS Code / Zed / JetBrains 集成 |

这些 `.exe` **不是从源码编译的**，而是由 `distlib`（setuptools 依赖）生成的模板化 launcher，内容固定为：
1. 硬编码指向 `.venv\Scripts\python.exe`
2. 硬编码入口模块和函数名
3. 运行时启动 Python 执行 `from <module> import <func>; <func>()`

每次 `uv sync` 或 `pip install -e .` 会重新生成这些 exe，`.venv/` 在 `.gitignore` 中，不会提交到 Git。

注意，CLI / TUI / Desktop 3种常用模式的启动入口都是 `hermes.exe` ，但是在**开发模式**下，可以跳过 `hermes.exe`: 
- CLI/TUI 直接调用 `python -m hermes_cli.main` 启动;
- Desktop 直接使用 `npm run dev --workspace apps/desktop` 启动。

`hermes desktop`（入口 `cmd_gui()`）内部流程：
1. 检查 `apps/desktop/package.json` 是否存在
2. 执行 `npm install`（如需要）→ `npm run build`（source 模式）或 `npm run pack`（packaged 模式）
3. 启动 Electron（source 模式：`electron .`；packaged 模式：直接运行 `release/win-unpacked/Hermes.exe`）


**开发环境中的子命令调用方式**（无需打包）：

```powershell
# 方式一：通过 .exe wrapper（和打包后完全一致）
.venv\Scripts\hermes.exe config
.venv\Scripts\hermes.exe model list
.venv\Scripts\hermes.exe setup

# 方式二：通过 Python 模块
.venv\Scripts\python.exe -m hermes_cli.main config
.venv\Scripts\python.exe -m hermes_cli.main model list
```

两种方式走相同的代码路径：`hermes_cli.main:main()` → argparse 解析子命令 → 分发到对应 handler。


---
### 三种启动方式的关系

```text
┌─ Desktop (Electron) ─────────────────────────┐
│ 独立窗口，React UI (@assistant-ui/react)      │
│ 后端: spawn python -m hermes_cli.main serve   │
│       (tui_gateway HTTP/WebSocket 子进程)     │
└──────────────────────────────────────────────┘

┌─ TUI (--tui) ─────────────────────────────────┐
│ 终端内，Ink/React UI (tsx/node 子进程)         │
│ 后端: 同进程 tui_gateway (JSON-RPC over stdio) │
└───────────────────────────────────────────────┘

┌─ CLI (默认) ─────────────────────────────────┐
│ 终端内，纯文本 (prompt_toolkit + rich)        │
│ 无分离前后端，全部在同一 Python 进程内         │
└─────────────────────────────────────────────┘
```

Desktop 和 TUI 共享同一套 `tui_gateway` JSON-RPC 协议，只是前端渲染层不同。


### 其他`hermes`子命令

（1）`hermes dashboard`子命令

它启动一个本地 FastAPI 服务器（Web Dashboard），浏览器访问 `localhost:xxxx`，内部嵌入了 TUI（通过 PTY bridge 把 `hermes --tui` 映射到 xterm.js）。


（2）`hermes serve`子命令

它是  `hermes dashboard` 的 **headless 无浏览器版本**的后端，Desktop 的 Electron 主进程正是 spawn `hermes serve` 子进程来启动python后端并提供 JSON-RPC 服务：

```powershell
# 仅启动后端服务（不打开浏览器），监听 127.0.0.1:9119
.venv\Scripts\python.exe -m hermes_cli.main serve --no-open
```

`hermes serve` 和 `hermes dashboard` 共享同一套 `start_server()` 函数，区别仅在于 `serve` 默认不打开浏览器。


---
### VS Code 调试配置

在 `.vscode/launch.json` 中添加：

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Hermes CLI",
            "type": "debugpy",
            "request": "launch",
            // "module": "hermes_cli.main",
            "program": "hermes",
            "cwd": "${workspaceFolder}",
            "console": "integratedTerminal",
            "env": {
                "HERMES_HOME": "${workspaceFolder}\\.hermes"
            }
        },
        {
            "name": "Hermes Agent (single run)",
            "type": "debugpy",
            "request": "launch",
            "program": "${workspaceFolder}\\run_agent.py",
            "cwd": "${workspaceFolder}",
            "console": "integratedTerminal",
            "env": {
                "HERMES_HOME": "${workspaceFolder}\\.hermes"
            }
        },
        {
            "label": "Hermes Desktop (dev)",
            "type": "npm",
            "script": "dev",
            "path": "apps/desktop",
            "problemMatcher": [],
            "options": {
                "env": {
                    "HERMES_HOME": "${workspaceFolder}\\.hermes"
                }
            }
        }
    ]
}
```

-----------------------------------------------------------------------
## Desktop 模式 FAQ

---
### Q1: Desktop模式启动脚本定义在哪？

**Desktop模式启动的 `npm run dev` 脚本在哪？根 `package.json` 里没有这个 key。**

根 `package.json` 的 `workspaces` 数组声明了子工作区 `"apps/*"`，所以 `apps/desktop` 是合法工作区。
`--workspace apps/desktop` 让 npm 去 `apps/desktop/package.json` 查找脚本，其中有：

```json
"dev": "concurrently -k \"npm:dev:renderer\" \"npm:dev:electron\""
```

即同时启动 Vite 前端 dev server（`:5174`）和 Electron 窗口。

---
### Q2: 如何关闭热更新？

Vite HMR 通过环境变量控制：

```powershell
$env:VITE_HMR = "false"; npm run dev --workspace apps/desktop
```

或手动构建后冷启动（完全无热更新）：

```powershell
cd apps/desktop
npm run build          # tsc + vite build → dist/
npx electron .         # Electron 直接加载 dist/
```

---
### Q3: Desktop(Electron) 模式下，Python 后端会自动启动吗？

会。Electron 主进程（`apps/desktop/electron/main.cjs`）在窗口就绪后自动 spawn Python 后端子进程。
后端解析优先级为 `HERMES_DESKTOP_HERMES_ROOT` → `SOURCE_REPO_ROOT`（仅开发模式）→ `ACTIVE_HERMES_ROOT`（从 `HERMES_HOME` 推导的 `$HERMES_HOME/hermes-agent`）。

实际 spawn 的命令是 `python -m hermes_cli.main serve`（即 `tui_gateway` 的 HTTP/WebSocket 版本），
前端通过 JSON-RPC over WebSocket 与它通信。关闭 Electron 窗口时后端自动终止，无需手动管理。

```
npm run dev
  ├── Vite (:5174) — 前端 HMR
  └── Electron 主进程
        └── spawn python -m hermes_cli.main serve (tui_gateway)
              └── AIAgent + 工具执行 + 模型调用
```

---
### Q4: `install.ps1` 默认会构建 Desktop 并创建快捷方式吗？

**不会。** Desktop 构建是**显式 opt-in** 的，只有传递 `-IncludeDesktop` 参数时才会触发。

具体来说：
- **普通 CLI 用户**运行 `irm https://hermes-agent.nousresearch.com/install.ps1 | iex` → **不会**构建 Desktop，不会创建快捷方式
- **通过 Hermes-Setup.exe（GUI 安装器）**安装时 → 会传递 `-IncludeDesktop`，构建 Desktop 并在 Start Menu 和 Desktop 创建 `.lnk` 快捷方式
- 快捷方式指向打包好的 `apps\desktop\release\win-unpacked\Hermes.exe`（Electron 二进制），**而非** `venv\Scripts\hermes.exe desktop`

源码中的注释明确说明了设计意图（`install.ps1` 第 46-58 行）：

> The canonical CLI one-liner (irm | iex) omits the flag too; terminal users don't need a desktop binary built for them, and `hermes desktop` already builds on demand.

---

### Q5: Electron快捷方式启动 vs `hermes desktop` 命令，效果一样吗？

**最终启动的桌面应用界面完全相同，但启动路径不同，且使用的数据/配置目录可能不同。**

| | 快捷方式（直接启动打包 exe） | `hermes desktop` |
|---|---|---|
| 启动路径 | Explorer 直接运行打包好的 Electron 应用 | Python `cmd_gui()` → 检查构建 → `subprocess.run([Hermes.exe])` |
| 构建开销 | 无（秒开） | 有内容哈希跳过机制，但首次/有变更时需完整构建 |
| `HERMES_HOME`解析 | Electron 自己调用 `resolveHermesHome()` 解析 | 继承当前终端 shell 的 `HERMES_HOME` 环境变量 |

> `hermes desktop` 底层最终也是打包Electron exe应用，然后在子进程中调用执行它。

关键差异在于 `HERMES_HOME` 的解析：

- `hermes desktop`：`cmd_gui()` 复制当前进程的 `os.environ`，传递给子进程里执行的 `Hermes.exe`，所以 `HERMES_HOME` 继承自当前终端的环境变量。

- 快捷方式启动：由 Explorer 直接启动，不经过任何终端。`resolveHermesHome()`（`main.cjs` 第 290-320 行）按以下优先级解析：
  1. `HERMES_HOME` 环境变量（如果有）
  2. Windows：从注册表读取 User 级别的 `HERMES_HOME`（`setx` 持久化的值）
  3. Windows 默认：`%LOCALAPPDATA%\hermes`
  4. macOS/Linux 默认：`~/.hermes`

这意味着：
- 如果使用`install.ps1`安装时通过 `[Environment]::SetEnvironmentVariable("HERMES_HOME", ..., "User")` **持久化**了 `HERMES_HOME`，两种方式完全等价，使用相同的配置/数据/venv。
- 如果只在终端会话中**临时**设置了 `$env:HERMES_HOME`，快捷方式启动时读不到它，会回退到 `%LOCALAPPDATA%\hermes`——这就是两者可能使用不同数据目录的情况。
- 后端 Python 进程（`python -m hermes_cli.main serve`）由 Electron spawn，继承 Electron 设置的环境变量，所以后端的 `HERMES_HOME` 始终一致。

---
### Q6: 如何生成类似 install.ps1 的独立 exe 启动入口？

`install.ps1 -IncludeDesktop` 本质上就是执行 `npm run pack`，产出一个独立的 `Hermes.exe`，完全可以手动完成同样的操作。

在开发环境中执行打包构建：

```powershell
cd apps/desktop
npm run pack
```

内部流程：

```powershell
npm run pack
  └── npm run build          # tsc 编译 + vite 打包 → dist/
  └── npm run builder -- --dir  # electron-builder 产出 win-unpacked/
```

产物位置：

```
apps/desktop/release/win-unpacked/Hermes.exe
```

与 `npm run dev` 的关键区别：

| | `npm run dev` | `npm run pack` 产物 |
|---|---|---|
| 前端加载 | Vite dev server 实时编译 | 预构建的 `dist/` 静态文件 |
| 热更新 | ✅ HMR | ❌ 无 |
| Electron | 开发版 electron | 打包后的 electron + 应用壳 |
| 启动速度 | 需等待 Vite 编译 | 秒开 |
| 用途 | 日常开发调试 | 模拟用户环境 / 创建快捷方式 |

打包后的 exe 需要能找到 Python 后端，推荐设置 `HERMES_DESKTOP_HERMES_ROOT`：

```powershell
[Environment]::SetEnvironmentVariable(
    "HERMES_DESKTOP_HERMES_ROOT",
    "D:\Path\to\hermes-agent",
    "User"
)
```

几点说明：
- 打包后的 `Hermes.exe` **不依赖系统 Node.js**。Electron 内嵌了自带的 Node.js 运行时（打包在 `win-unpacked/` 中），启动时完全自包含。
- 而且`electron-builder`在打包时会将Electron的二进制文件（`electron.exe`重命名为`Hermes.exe`）和`node_modules`一起放进 `win-unpacked`。
- 系统 Node.js 仅在 `npm run pack` 构建阶段使用（`tsc`、`vite`、`electron-builder`），产出的 exe 与系统 Node.js 无关。
- 后端是 Python 进程（`python -m hermes_cli.main serve`），由 `HERMES_DESKTOP_HERMES_ROOT` 指向的 venv 中的 Python 解释器运行，也不需要 Node.js。

---
### Q7: 打包后的 exe 和项目文件夹移动到其他位置还能运行吗？

可以，但需要同时设置两个环境变量指向新位置。

`main.cjs` 中后端解析优先级为：

```
HERMES_DESKTOP_HERMES_ROOT（最高）
  → SOURCE_REPO_ROOT（仅开发模式）
  → ACTIVE_HERMES_ROOT（从 HERMES_HOME 推导）
```

假设项目从 `D:\old\hermes-agent` 移动到 `E:\new\hermes-agent`：

```powershell
# 指向新位置的项目根目录（Desktop 用来找 Python 后端和 venv）
[Environment]::SetEnvironmentVariable(
    "HERMES_DESKTOP_HERMES_ROOT",
    "E:\new\hermes-agent",
    "User"
)

# 指向新位置的 .hermes 数据目录（配置/日志/session）
[Environment]::SetEnvironmentVariable(
    "HERMES_HOME",
    "E:\new\hermes-agent\.hermes",
    "User"
)
```

注意事项：
- `Hermes.exe` 不是单文件，依赖同目录下的 `resources/app.asar`，移动时必须将整个 `win-unpacked/` 目录一起移动。
- `APP_ROOT` 和 `SOURCE_REPO_ROOT` 是构建时 baked-in 的路径，但打包模式下它们不影响后端解析。
- 启动后检查 `$HERMES_HOME\logs\` 下的日志确认后端是否正常连接。


-----------------------------------------------------------------------
## 补充说明

### 未安装的可选组件

以下组件在 `install.ps1` 中会被安装，但开发环境不需要：

| 组件 | 影响 | 手动安装命令 |
|---|---|---|
| ripgrep (`rg`) | `search_files` 回退到 `findstr`，速度较慢 | `winget install BurntSushi.ripgrep.MSVC` |
| ffmpeg | TTS 语音消息不可用 | `winget install Gyan.FFmpeg` |
| agent-browser + Chromium | `browser_navigate` 等浏览器工具不可用 | `npx playwright install chromium` |


### 与 install.ps1 的关键差异

| 项目 | install.ps1 | 本方案 |
|---|---|---|
| UV | 下载独立 UV 到 `%LOCALAPPDATA%\hermes\bin\` | 使用系统已有 UV |
| Git | 下载 PortableGit 到 `%LOCALAPPDATA%\hermes\git\` | 使用系统已有 Git |
| Node.js | 下载便携版到 `%LOCALAPPDATA%\hermes\node\` | 使用系统已有 Node.js |
| 源码 | clone 到 `%LOCALAPPDATA%\hermes\hermes-agent\` | 当前仓库即源码 |
| HERMES_HOME | `%LOCALAPPDATA%\hermes` | 仓库内 `.hermes\` |
| PATH 修改 | 将 `$HermesHome\bin` 和 `venv\Scripts` 写入 User PATH | 不修改 PATH |
| ripgrep/ffmpeg | 通过 winget 安装 | 跳过 |


### 关键文件速查

| 文件 | 作用 |
|---|---|
| `hermes_constants.py` — `get_hermes_home()` | 所有路径查找的入口，读取 `HERMES_HOME` 环境变量 |
| `hermes_cli/main.py` — `_launch_tui()` | TUI 启动逻辑，spawn Node.js 子进程 |
| `hermes_cli/main.py` — `_make_tui_argv()` | TUI 的 Node 命令行构建，`--dev` 走 tsx |
| `hermes_cli/main.py` — `cmd_gui()` | `hermes desktop` 命令 handler，构建并启动 Electron |
| `hermes_cli/main.py` — `cmd_dashboard()` | `hermes dashboard` / `hermes serve` 命令 handler |
| `hermes_cli/subcommands/gui.py` | `hermes desktop` 子命令 argparse 定义 |
| `hermes_cli/web_server.py` — `start_server()` | Dashboard / serve 的 HTTP + WebSocket 服务器 |
| `apps/desktop/package.json` — `dev` script | Desktop 开发模式：Vite + Electron 并发 |
| `apps/desktop/electron/main.cjs` | Electron 主进程，spawn Python 后端、窗口管理、自动更新 |
| `pyproject.toml` — `[project.scripts]` | 三个控制台入口：`hermes` / `hermes-agent` / `hermes-acp` |
| `pyproject.toml` — `[project.optional-dependencies] dev` | 开发依赖清单 |
| `scripts/run_tests.sh` | 测试入口（需 Git Bash），设置 TZ=UTC 等 CI 一致环境 |


### 定制化部署

此文档中总结的部署操作步骤完全可以用于定制化部署Hermes，代替 `install.ps1`，控制Hermes的部署路径和数据目录。

Hermes 的核心工作方式就是：**`HERMES_HOME` 决定一切**。
只要设置了这个变量，Python 代码中所有路径都通过 `get_hermes_home()` 解析，所以把数据目录放在哪都行。

更新Hermes的流程，也只需要拉取main分支最新代码，对于Desktop模式重新执行Electron应用构建过程即可。

`hermes update`命令执行的也是上述操作（参考 `cmd_update` 函数，`main.py` 第 9064 行起）。

此定制化部署方案有如下优势：
- 完全控制安装路径：`HERMES_HOME` 可以指向任意目录，不影响系统环境
- 不修改`PATH`：不会污染系统环境变量
- 复用已有工具链：使用系统已有的 UV、Git、Node.js
- 更新简单：`git pull` + `uv sync` + `npm ci` + 重新构建 Desktop 即可
- **多实例共存**：可以同时维护多个不同版本的开发环境（通过不同的 `HERMES_HOME` 和 checkout 目录）
