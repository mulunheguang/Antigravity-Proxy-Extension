# Antigravity Proxy Extension for macOS 🚀

<p align="center"><b>为 Antigravity App / IDE 提供透明代理的 macOS 插件</b><br />图形化配置 · SOCKS5 上游 · 动态库注入 · SNI 中继 · 环境诊断</p>

> 基于 [raybz/Antigravity-Proxy](https://github.com/raybz/Antigravity-Proxy) 修改，针对新版 macOS Antigravity 的应用结构与代理连接问题进行兼容性修复。原项目的核心方案和大部分代码来自原作者，本仓库保留其署名与 MIT 许可证。

---

## 📖 项目介绍

Antigravity 的部分网络请求可能不遵循 macOS 系统代理，仅开启代理客户端的“系统代理”后，仍可能出现登录、模型请求或语言服务连接失败。

本插件通过 **动态库注入、hosts 映射与 TLS SNI 中继**，把相应连接转发到你已有的本地 **SOCKS5 代理**，并在编辑器内提供配置、启动、停止和诊断入口。

它不创建 TUN/TAP 虚拟网卡，也不提供代理节点。需要先运行 Shadowrocket、Clash 等代理客户端，并开启可用的无认证 SOCKS5 监听。

- **本仓库：** [mulunheguang/Antigravity-Proxy-Extension](https://github.com/mulunheguang/Antigravity-Proxy-Extension)
- **上游原项目：** [raybz/Antigravity-Proxy](https://github.com/raybz/Antigravity-Proxy)
- **独立启动器版：** [Antigravity-Proxy-Launcher](https://github.com/mulunheguang/Antigravity-Proxy-Launcher)，适合希望通过 macOS 图标启动、不依赖编辑器插件界面的用户。

### 代码来源与修改原因

本仓库是原项目的修改版本，不是从零开发的独立代理实现。动态库、FakeIP、SNI 中继、扩展配置页面、签名与特权辅助流程均以原项目为基础；本分支的改动由 mulunheguang 整理维护，不代表原作者发布或背书。

修改的直接原因是：所使用的新版 Antigravity 主程序为 `Contents/MacOS/Antigravity`，而原有逻辑依赖 `Contents/MacOS/Electron`，导致有效的 `.app` 被判定为“该路径不是有效的 Antigravity.app”，并影响后续签名、启动和进程检测。

相对于本仓库所基于的上游版本，主要调整包括：

- **应用路径兼容：** 从 `Info.plist` 的 `CFBundleExecutable` 读取实际主程序名，统一用于校验、诊断、启动和停止；同步调整 Makefile 与辅助脚本。
- **签名兼容：** 兼容新旧语言服务文件位置，并调整签名检查和执行顺序。无需把主程序重命名为 `Electron`。
- **注入修复：** 使用 Mach-O interpose 机制处理 `connect` / `getaddrinfo`，修复非阻塞连接和 IPv4 映射 IPv6 场景下的代理握手问题。
- **协议校验：** 明确注入库和中继需要无认证 SOCKS5；拒绝仅 HTTP 的配置，并处理握手应答分片、提前断开等情况。
- **打包检查：** 使用 VSIX 打包流程检查内置二进制，面向 Apple Silicon 分发。

这些说明针对本分支的修改背景，不表示上游后续版本仍存在同样问题。

---

## ✨ 功能特性

- 在编辑器中配置代理地址、端口与目标 `.app` 路径。
- 启动目标 Antigravity，并准备 hosts 与 SNI 中继。
- 管理注入所需签名，提供停止、清理和完全停用入口。
- 提供 `AG-Proxy` 状态栏、配置校验、环境诊断和日志。
- 可选免密 sudo helper，以及扩展激活后的自动准备功能。

### App 和 IDE 都能使用吗？

可以面向 **Antigravity App 和 Antigravity IDE** 使用，但需要区分两个角色：

- **插件宿主：** 安装并运行 VSIX 的编辑器，例如 VS Code、Cursor，或支持该 VSIX 的 Antigravity IDE。
- **代理目标：** 配置中指定的 Antigravity `.app`，可以是 App 或 IDE 的实际安装路径。

hosts 与中继对配置的域名在系统层面生效，因此运行中的 App 和 IDE 都可能使用同一条中继链路；动态库注入则取决于目标进程的启动方式及签名。并非所有应用版本、模型、域名或协议都已验证。

如果将插件装在目标 IDE 自身中，执行停止、重签名或完全停用可能影响当前 IDE。首次配置建议在 VS Code / Cursor 中完成，并先保存工作。

---

## 🚀 快速开始

### 1. 准备环境

- **Apple Silicon Mac（arm64）**：当前仓库内置二进制为 arm64；不承诺 Intel Mac 可直接使用。
- 已安装 Antigravity，以及支持 VSIX 的本地桌面编辑器；扩展声明要求 VS Code API `^1.80.0`。
- 已启动本地无认证 SOCKS5 代理，并确认它能访问所需服务。
- 安装 Xcode Command Line Tools（已安装可跳过）：

```bash
xcode-select --install
```

### 2. 获取并安装 VSIX

如果本仓库的 [Releases](https://github.com/mulunheguang/Antigravity-Proxy-Extension/releases) 已提供安装包，下载对应的 `darwin-arm64.vsix`；没有发布包时，按下方“从源码打包”生成。

插件可以安装在 **Cursor、VS Code，或支持 VSIX 扩展安装的 Antigravity IDE（反重力 IDE）** 中，选择其中一个即可。这里指的是带有扩展管理功能的 IDE；独立 Antigravity App 是代理目标，不是本步骤的插件安装入口。

以下步骤在你选择的 **Cursor / VS Code / Antigravity IDE** 中操作：

1. 按 `Cmd + Shift + P` 打开命令面板。
2. 选择 **Extensions: Install from VSIX…（扩展：从 VSIX 安装）**。
3. 选择生成的 `.vsix` 文件，例如 `antigravity-proxy-0.0.7-darwin-arm64.vsix`。
4. 按编辑器提示重载窗口。

这里选择的是 **VSIX 安装包**；目标 `Antigravity.app` 在下一步的配置页中填写。

当前沿用上游扩展标识 `raybz.antigravity-proxy`，安装本分支 VSIX 会使用同一扩展身份。该标识用于兼容既有配置，不代表这是原作者发布的安装包；请确认安装包来自本仓库或自行构建。

### 3. 配置代理与应用路径

在命令面板搜索 **Antigravity Proxy**，打开 **⚙️ 打开配置页面**，填写：

- **代理地址：** 通常为 `127.0.0.1`。
- **代理端口：** 你的代理客户端实际开启的 SOCKS5 端口。插件默认值为 `10808`，不一定适合你的客户端。
- **代理类型：** `SOCKS5`。
- **Antigravity 路径：** 例如 `/Applications/Antigravity.app`。填写 `.app` 本身，不要填写包内的可执行文件或项目目录。

留空时会尝试 `/Applications/Antigravity.app` 和 `~/Applications/Antigravity.app`。若 App 与 IDE 分开安装，请填写需要启动的那一份 `.app` 的完整路径。

点击 **校验配置**，通过后点击 **保存配置**。

> **Shadowrocket 用户：** 不要仅凭“HTTP 127.0.0.1:1082”就判断可用。如果同一端口也支持无认证 SOCKS5，可填写 `1082` 并选择 SOCKS5；如果只支持 HTTP，需要在客户端开启 SOCKS5 或支持 SOCKS5 的混合监听，并填写实际端口。端口号没有统一固定值。

### 4. 首次签名与启动

1. 保存工作并退出准备代理的目标 Antigravity。
2. 首次使用或应用更新后，执行 **🔑 强制重签名**，按提示授权并等待完成。
3. 执行 **🚀 启动代理**。流程会准备 hosts、启动中继，再注入并启动目标程序。
4. 未安装免密 helper 时，根据终端或系统提示输入管理员密码。
5. 打开 **📊 环境诊断与状态**，再在 Antigravity 中测试实际登录或模型请求。

**签名流程会修改目标应用的 `Info.plist` 环境配置并进行本地重签名。** 不需要重命名主程序。应用更新后可能需要再次签名；不希望使用此注入方式的用户可以查看独立启动器版。

`AG-Proxy` 显示绿色意味着当前实现检查的 hosts、中继进程、目标主进程、内置文件与 SOCKS5 协商条件通过，**不等于每个业务请求已成功通过代理**。

---

## ⚙️ 如何修改配置

打开命令面板 → **Antigravity Proxy: ⚙️ 打开配置页面** → 修改 → **校验配置** → **保存配置**。

修改代理地址或端口后，需重新执行 **准备特权环境** 来重启中继；为使已经运行的注入进程也读取新设置，建议保存工作后 **停止代理，再启动代理**。只保存配置不会自动替换正在运行的中继参数。

也可以修改编辑器的 `settings.json`。以下示例中的 `1082` 必须替换为你实际可用的 SOCKS5 端口：

```json
{
  "antigravity-proxy.proxyHost": "127.0.0.1",
  "antigravity-proxy.proxyPort": 1082,
  "antigravity-proxy.proxyType": "socks5",
  "antigravity-proxy.timeout": 5000,
  "antigravity-proxy.antigravityAppPath": "/Applications/Antigravity.app",
  "antigravity-proxy.autoStart": false,
  "antigravity-proxy.autoPrepareHostsRelay": true,
  "antigravity-proxy.showStatusBar": true
}
```

配置页会尝试同步用户、当前工作区和文件夹作用域的设置，以避免工作区覆盖导致修改不生效。插件运行时生成 `/tmp/antigravity-config.yaml`，不要把这个临时文件作为长期配置入口；仓库根目录的 `config.yaml` 用于 Makefile 命令行流程。

### 自动启动与免密模式

- `autoStart` 默认关闭：开启后，在**插件宿主激活扩展时**执行完整启动流程，不是 macOS 开机服务。
- `autoPrepareHostsRelay` 默认开启：仅当已安装免密 helper、未开 `autoStart`、且未主动完全停用代理时，按需准备 hosts / 中继；不负责启动目标应用。
- 可选执行 **🔐 一次性安装免密 sudo**，它会安装 `/usr/local/bin/antigravity-proxy-helper` 并写入 `/etc/sudoers.d/antigravity-proxy`。这会授予 helper 相应免密特权；首次安装仍需授权。
- 扩展更新后若提示 helper 过期，重新执行安装命令同步 helper。

此插件版的配置入口是编辑器中的 **AG-Proxy / 命令面板**；macOS 顶部菜单栏的 **AG** 属于独立启动器版。

---

## 🛑 停止、清理与卸载

- **⏹ 停止代理：** 结束配置的 Antigravity 主进程并清理 hosts / 中继，不等于移除全部持久注入配置。
- **🧹 清理 hosts 与中继：** 清理代理环境，不主动结束目标应用。
- **🔕 完全停用代理：** 关闭自动拉起设置、结束目标进程、清理 hosts / 中继、移除相关 `LSEnvironment` 项并重签名。

卸载扩展前，先保存工作，执行 **完全停用代理**，确认清理完成，再卸载。只退出应用或卸载 VSIX 不保证撤销 hosts 和注入配置。

“完全停用”恢复的是此扩展涉及的代理设置，**不会恢复厂商原始代码签名**，也不会撤销你另外配置的系统代理。若要恢复厂商原始应用包，请重新安装官方应用。

如果此前安装过免密 helper，完全停用后不再需要它时，可单独移除：

```bash
sudo rm -f /etc/sudoers.d/antigravity-proxy
sudo rm -f /usr/local/bin/antigravity-proxy-helper
```

---

## 🔍 检测与常见问题

### TCP 连通、SOCKS5 握手和实际请求有什么区别？

TCP 连通只说明端口有人监听；SOCKS5 握手通过只说明对方接受无认证协议协商。要确认上游代理能转发 HTTPS，还需要建立隧道并完成实际请求。例如：

```bash
curl --noproxy '' --proxy socks5h://127.0.0.1:1082 \
  --connect-timeout 10 --max-time 30 \
  -o /dev/null -sS -w 'HTTP %{http_code}\n' \
  https://www.googleapis.com/
```

替换为实际端口。此命令保留 TLS 证书校验；成功完成请求后，即使服务根路径返回 HTTP 404，也说明这一次 HTTPS 请求通过上游完成，但不代表 Antigravity 的登录、账号权限或所有模型请求都正常。该命令验证的是**上游 SOCKS5**，不验证本地 SNI 中继或应用注入。

### 仍然提示应用路径无效

确认填写的是完整 `.app` 路径，且其中 `Info.plist` 的 `CFBundleExecutable` 指向存在、可执行的主程序。本分支不再要求文件名必须是 `Electron`，不要手动重命名应用内的文件。

### 中继启动失败、状态一直不绿

先检查代理协议与端口，再确认本机 `443` 没有被其他服务占用。避免与独立启动器版同时启动两套中继；切换版本前先清理上一套代理环境。

- 扩展日志：命令面板 → **📋 查看日志**。
- 中继日志：`/tmp/antigravity-relay.log`。
- 中继 PID 文件：`/tmp/antigravity-relay.pid`。

### App 能用，IDE 或某个模型不能用

先确认目标路径及实际请求域名。hosts / SNI 中继仅覆盖下方列出的域名，动态库也不能保证拦截所有网络栈。一次聊天成功或状态绿色都不能单独证明流量路径；需结合日志及实际请求排查。

---

## 🧩 工作原理与范围

```text
目标域名 → hosts 映射到 127.0.0.1
         → 本地 :443 TLS SNI 中继
         → SOCKS5 上游 → 目标服务

可注入进程 → libantigravity.dylib
           → connect / getaddrinfo 拦截与 FakeIP
           → SOCKS5 上游 → 目标服务
```

hosts / 中继当前覆盖以下域名：

```text
daily-cloudcode-pa.googleapis.com
cloudcode-pa.googleapis.com
oauth2.googleapis.com
accounts.google.com
www.googleapis.com
generativelanguage.googleapis.com
content-cloudcode-pa.googleapis.com
```

hosts 是系统配置，访问这些域名的其他程序也可能受影响。中继依赖 TLS SNI；本方案不等于覆盖所有应用、所有域名、UDP 或 QUIC 的全局 VPN。受 SIP 等保护的系统程序也不能保证可注入。

---

## 🛠 从源码打包

在 Apple Silicon Mac 上安装 Xcode Command Line Tools、Node.js 与 npm，然后执行：

```bash
git clone https://github.com/mulunheguang/Antigravity-Proxy-Extension.git
cd Antigravity-Proxy-Extension

# 重新编译本机二进制；只构建，不启动或重签目标应用
make -B build
cp bin/libantigravity.dylib bin/antigravity-relay vscode-extension/bin/

cd vscode-extension
npm ci
npm run compile
npx --package=@vscode/vsce vsce package --target darwin-arm64
```

生成的 `.vsix` 位于 `vscode-extension/`。打包前会再次执行 TypeScript 编译及内置二进制检查。生成安装包不等于发布到扩展市场。

### 命令行入口

仓库也保留上游的 Makefile 工作流。编辑根目录 `config.yaml`，填写实际 SOCKS5 地址与端口，再使用：

```bash
make build      # 仅构建
make sign       # 配置注入环境并按需签名
make run        # 准备 hosts / 中继并注入启动
make resign     # 强制重新配置签名
make help       # 查看其他命令
```

`make` 默认会进入运行流程，不要把它当作仅编译。非默认安装路径可通过 `make run ANTIGRAVITY_APP='/Applications/你的应用.app'` 指定。Makefile 与插件使用不同的配置入口，日常使用建议选定一种方式管理。

---

## 📁 项目结构

```text
Antigravity-Proxy-Extension/
├── src/                         # 动态库、SOCKS5、FakeIP 与 SNI 中继
├── bin/                         # 原生二进制
├── config.yaml                  # 命令行模式配置
├── Makefile                     # 构建、签名与命令行启动
├── LICENSE                      # 保留上游 MIT 版权声明
└── vscode-extension/
    ├── src/                     # 插件界面与代理管理逻辑
    ├── scripts/                 # 特权 helper 与打包检查
    ├── bin/                     # 随 VSIX 分发的二进制
    ├── package.json            # 扩展配置与命令声明
    └── README.md                # 扩展详情页说明
```

---

## 🙏 致谢与许可证

感谢 [raybz](https://github.com/raybz) 提供 [Antigravity-Proxy](https://github.com/raybz/Antigravity-Proxy) 的核心实现。本仓库保留原项目的代码来源说明及 `Copyright (c) 2026 ray` 声明，使用 [MIT License](LICENSE)。修改与再分发请保留相应版权和许可文本。

上游文档还注明方案概念参考了 [yuaotian/antigravity-proxy](https://github.com/yuaotian/antigravity-proxy) 的 Windows 实现，亦在此保留致谢。
