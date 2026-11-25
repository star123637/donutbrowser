# Donut Browser 在 Windows 上的源码部署

本记录对应上游稳定版 `v0.36.0`，提交为 `61092ed19af08abaed20158708e1cb3686e284ea`。构建和部署文档分别使用标签检出和 fork 的 `main` 检出，以免把编译产生的文件带入文档提交。[发布页](https://github.com/donutbrowser/donutbrowser/releases/tag/v0.36.0) · [贡献指南](https://github.com/donutbrowser/donutbrowser/blob/v0.36.0/CONTRIBUTING.md) · [发布流水线](https://github.com/donutbrowser/donutbrowser/blob/v0.36.0/.github/workflows/release.yml)

## 环境和依赖

| 项目 | v0.36.0 的要求或选型 | 本机记录 |
| --- | --- | --- |
| 操作系统 | Windows x64 | Windows 11 家庭版，系统构建 26200 |
| Git | 用于检出发布标签 | 2.49.0.windows.1 |
| Node.js | 仓库 `.node-version` 指定 26 | 项目专用 26.13.1，系统另有 22.21.1 |
| pnpm | `package.json` 指定 12.16.1 | 项目专用 12.16.1，系统另有 10.25.0 |
| Rust 与 Cargo | stable，目标 `x86_64-pc-windows-msvc` | 见本次验证记录 |
| MSVC 和 Windows SDK | Visual Studio Build Tools 的“使用 C++ 的桌面开发”工作负载 | 见本次验证记录 |
| WebView2 | Tauri 桌面窗口使用 Evergreen Runtime | 158.0.4370.33 |
| Tauri | 项目本地 CLI，Rust 依赖由 Cargo.lock 锁定 | CLI 2.12.4，Rust crate 2.12.5 |
| CMake | Windows 原生依赖使用 | 4.0.0 |

Tauri 使用 WebView2 绘制应用窗口；用户新建的浏览器环境由 Wayfern 启动。Windows 依赖声明启用了 `aws-lc-sys` 的 `prebuilt-nasm`。WinFsp 用于临时配置的内存卷，应用按需加载其驱动。[Tauri Windows 前置条件](https://v2.tauri.app/start/prerequisites/#windows) · [项目依赖](https://github.com/donutbrowser/donutbrowser/blob/v0.36.0/src-tauri/Cargo.toml)

## 复现源码构建

在 PowerShell 中从有足够空间的工作目录检出发布标签。下面的路径与本次构建一致。

```powershell
git clone --config core.autocrlf=false --branch v0.36.0 --single-branch --depth 1 https://github.com/star123637/donutbrowser.git E:\src\donutbrowser-v0.36.0
Set-Location E:\src\donutbrowser-v0.36.0
git rev-parse HEAD
```

本次通过项目外的 npm 前缀部署 Node 和 pnpm。在新开的 PowerShell 中重新设置 PATH，即可继续使用这组版本。

```powershell
$toolsDir = 'E:\src\donut-windows-tools'
npm.cmd install --prefix $toolsDir --registry=https://registry.npmjs.org --no-fund --no-audit node@26.13.1 pnpm@12.16.1
$env:Path = "$toolsDir\node_modules\node\bin;$toolsDir\node_modules\@pnpm\exe.win32-x64;$env:USERPROFILE\.cargo\bin;$env:Path"
node --version
pnpm.exe --version
```

安装 MSVC x64/x86 工具和 Windows SDK 后，使用 stable MSVC 工具链。可用下列命令检查工具链并执行上游的开发入口。首次编译还会构建 `donut-proxy`、下载并校验 Xray sidecar。

```powershell
rustc -vV
cargo --version
pnpm.exe install --frozen-lockfile --registry=https://registry.npmjs.org
pnpm.exe tauri dev
```

开发模式验收应能看到 Donut 原生窗口及其配置列表。开发服务器默认使用 `localhost:12341`。退出开发模式后，从同一标签目录生成 NSIS 安装包。

```powershell
$env:GITHUB_REF_NAME = 'v0.36.0'
pnpm.exe tauri build --bundles nsis
Get-ChildItem .\src-tauri\target\release\bundle\nsis\*.exe | Get-FileHash -Algorithm SHA256
```

## 本次验证记录

2026-11-25 在 Windows x64 上完成 fork、克隆、标签核对、锁文件依赖安装，以及项目本地 Tauri CLI 的版本检查。前端与打包脚本的 Node 测试子集运行 286 项，285 项通过，1 项按测试自身条件跳过。该批次排除了许可证生成测试和 Rust 测试。

初次检出时系统的 `core.autocrlf=true` 把 Linux AppStream 测试样本转换成 CRLF，导致一个断言失败。按 Git 中的原始字节恢复该 XML 后复测通过，样本的 Git blob 和标签一致。上面的克隆命令直接保留 LF。

## 浏览器环境的验收边界

v0.36.0 创建和启动环境时只接受 Wayfern（Chromium）。Firefox 系的 Camoufox 在 v0.26.0 开始弃用，v0.28.0 完全弃用。当前标签无法创建所要求的 Firefox 环境。[创建环境的校验](https://github.com/donutbrowser/donutbrowser/blob/v0.36.0/src-tauri/src/api_server.rs#L1797-L1809) · [启动校验](https://github.com/donutbrowser/donutbrowser/blob/v0.36.0/src-tauri/src/browser_runner.rs#L1518) · [上游变更记录](https://github.com/donutbrowser/donutbrowser/blob/v0.36.0/CHANGELOG.md)

本机验收改用用户选定的两个 Wayfern 环境。指纹对比要让两个环境分别访问同一个检测页，保存同一字段的结果。存储隔离需要单独验证：在同一来源中，让 A 写入一次性 cookie 和 localStorage 值，检查 B 的初始状态，再让 B 写入不同值并返回 A 复查。检测页的指纹哈希不同，本身不足以证明存储隔离。[CreepJS](https://abrahamjuliot.github.io/creepjs/) · [BrowserLeaks](https://browserleaks.com/)

## 发布包核验和再分发

GitHub Release 的 Windows x64 安装程序为 `Donut_0.36.0_x64-setup.exe`。下载文件共 27,914,970 字节，实测 SHA-256 为 `f5a8f769864018d590bb836e782517c928c7fdf6ee89ab33f7833b8332963cf5`，与 GitHub Release 资产摘要一致；Authenticode 状态为 `NotSigned`。本次按用户选择仅继续源码方案。

对外分发修改版时，按 AGPL-3.0 提供对应版本的完整源码、构建脚本和许可文本，保留版权说明。修改后的程序通过网络向用户提供服务时，也要依第 13 节向远程用户提供获取对应源码的途径。第三方引擎和组件分别适用其许可。[项目许可证](https://github.com/donutbrowser/donutbrowser/blob/v0.36.0/LICENSE)
