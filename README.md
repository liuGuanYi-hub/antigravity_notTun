# Google Antigravity 中国大陆网络连接解决方案

这是一个面向 Windows 用户的、进程级代理启动方案。

它适用于在中国大陆网络环境下遇到以下问题的 Google Antigravity 用户：

- Antigravity 主界面可以打开，但登录、模型或 Agent 无法连接；
- Electron 界面能够联网，但 Language Server / Agent 无法联网；
- 不希望开启 v2rayN Global Mode；
- 不希望启用 TUN 虚拟网卡；
- 不希望修改 Windows 持久化环境变量或整机代理策略。

本项目不是 Antigravity 汉化项目，也不是代理软件本身。它只提供一个独立启动入口，把 HTTP 代理环境变量临时注入 Antigravity 进程树。

> 重要声明：本方案不用于解除 Google 官方地区限制、绕过账号资格审查，也不保证所有网络请求都一定会被代理。它只解决遵循这些环境变量的程序和网络组件的连接问题。请自行确认网络服务、账号和当地法律要求。

## 0. 本机实测要点（2026-10-22，Windows 11 build 26100）

这份仓库是 `liuGuanYi-hub/antigravity_notTun`，上游为 `Molly1116/Antigravity-China-Access`（MIT，保留原作者那条 commit）。以下三条是在本机把方案替换掉 TUN 之后验证得到的，按踩坑概率排序：

1. **端口对不上就是白装。** `Start-Antigravity-Proxy.ps1` 在找不到 `proxy-config.psd1` 时静默回落到 `http://127.0.0.1:10808`，不报错、不提示，看上去装好了其实什么都没做。本机 HTTP 代理监听是 `127.0.0.1:7897`，所以这份仓库直接跟进了填好端口的 `proxy-config.psd1`（上游把它列进 `.gitignore`，这里改为跟踪，clone 即用）。装之前先 `netstat -ano | findstr :7897` 确认监听还在。
2. **根因：Language Server 是 Go 二进制，只读环境变量、不读系统代理注册表。** `%LOCALAPPDATA%\Programs\Antigravity\resources\bin\language_server.exe` 里能查到 `net/http.ProxyFromEnvironment`、`net/http.envProxyFunc` 符号和 `HTTP_PROXY` 字面量。Go 不会去读 WinINET，所以在 `ProxyEnable=1`、系统代理已经指向 7897 的情况下，Electron 界面照常联网、Agent 仍然直连失败。这正是"只有 TUN 能兜住"的原因，也正是往进程环境里注变量就能顶替 TUN 的原因。
3. **必须完全退出 Antigravity 再点快捷方式。** Antigravity 是单实例：旧进程还活着（包括托盘）时，新的启动只是把它唤回前台，环境变量进不去，表现和"这方案没用"一模一样。验证前先确认任务管理器里没有 `Antigravity` 和 `language_server` 进程。

## 1. 原理

启动链如下：

```text
桌面快捷方式
    -> wscript.exe（隐藏窗口）
    -> Start-Antigravity-Proxy.vbs
    -> Start-Antigravity-Proxy.ps1
    -> 官方 Antigravity.lnk 或官方 Antigravity.exe
    -> Antigravity 子进程继承代理环境变量
```

PowerShell 启动脚本在自己的进程环境中设置：

```text
HTTP_PROXY=http://127.0.0.1:10808
HTTPS_PROXY=http://127.0.0.1:10808
http_proxy=http://127.0.0.1:10808
https_proxy=http://127.0.0.1:10808
NO_PROXY=localhost,127.0.0.1,::1
no_proxy=localhost,127.0.0.1,::1
```

随后脚本启动官方 Antigravity。Windows 子进程通常会继承父进程环境，因此遵循 `HTTP_PROXY` / `HTTPS_PROXY` 的 Language Server、Agent 或网络库可以使用本地代理。

这些变量只存在于本次启动链的进程环境中：

- 不使用 `setx`；
- 不写入用户或系统环境变量；
- 不修改 WinINET；
- 不修改 WinHTTP；
- 不修改注册表、路由、DNS、TUN 或 v2rayN 配置；
- 不修改 Antigravity 安装目录。

不遵循这些变量的程序仍可能直连。因此本项目不承诺拦截所有流量。

## 2. 与 Global Mode / TUN 的区别

| 项目 | 本项目 | Global Mode | TUN |
| --- | --- | --- | --- |
| 作用范围 | Antigravity 启动进程树 | 通常是代理客户端的全局分流范围 | 虚拟网卡接管范围内的流量 |
| 是否修改整机网络 | 否 | 可能会 | 会改变路由/网络栈行为 |
| 是否修改持久化环境变量 | 否 | 通常不需要 | 通常不需要 |
| 是否影响其他软件 | 不应影响未继承该进程树的程序 | 可能影响 | 可能影响 |
| 依赖 | 应用和网络库支持 HTTP 代理变量 | 代理客户端规则 | TUN 驱动和路由 |

本项目的目标是：

```text
Antigravity 进程树 -> 127.0.0.1:10808 -> 本地代理
```

而不是：

```text
整台 Windows 电脑 -> VPN / Global Mode
```

## 3. 准备条件

1. Windows 10 或 Windows 11。
2. 已安装 Google Antigravity。
3. 已有一个本地 HTTP 代理端口。
4. 该端口能够接受 HTTP `CONNECT` 请求。

例如，v2rayN 的 Mixed/HTTP 监听端口可能是：

```text
http://127.0.0.1:10808
```

如果本地端口只有纯 SOCKS5 协议，请先使用代理客户端提供的 HTTP/Mixed 入口。不要直接把 `socks5://` 填到本项目的 `ProxyUrl`，除非你确认目标网络库支持该协议。

## 4. 配置代理地址和端口

本仓库已经把可用的 `proxy-config.psd1` 直接提交进版本控制，clone 下来就在，不需要复制示例：

```powershell
@{
    ProxyUrl = 'http://127.0.0.1:7897'
    NoProxy = 'localhost,127.0.0.1,::1'
}
```

换机器时通常只需要改 `ProxyUrl`。推荐使用本机无认证代理监听器，不要把代理账号、密码或令牌提交到 GitHub。

`proxy-config.example.psd1` 只作参考。**不要执行 `Copy-Item .\proxy-config.example.psd1 .\proxy-config.psd1`** —— 那会把已提交的真实端口覆盖回示例值，等于把第 0 节那个坑亲手装回去。若 `proxy-config.psd1` 缺失，启动脚本静默使用 `http://127.0.0.1:10808`。

## 5. 安装项目

本仓库没有发布 Release 压缩包，请直接克隆：

```powershell
git clone https://github.com/liuGuanYi-hub/antigravity_notTun.git
Set-Location .\antigravity_notTun
```

配置文件已随仓库提交，端口不同就直接编辑：

```powershell
notepad .\proxy-config.psd1
```

项目不需要安装 Go、Node、Python 或第三方运行库。脚本使用 Windows 自带的 PowerShell、Windows Script Host 和 Shell 快捷方式接口。

## 6. 创建桌面快捷方式

在项目目录中执行：

```powershell
powershell.exe -NoProfile -ExecutionPolicy Bypass -File .\New-Antigravity-ProxyShortcut.ps1
```

脚本默认创建：

```text
桌面\Antigravity (Proxy).lnk
```

它不会覆盖同名快捷方式。快捷方式的目标是隐藏运行的 `wscript.exe`，图标优先使用官方 `Antigravity.exe` 的图标。

## 7. 一键启动

双击桌面上的：

```text
Antigravity (Proxy).lnk
```

启动时不会打开长期驻留的黑色终端窗口。Antigravity 仍然使用官方安装文件和官方用户数据目录。

也可以直接运行：

```powershell
wscript.exe //B //NoLogo .\Start-Antigravity-Proxy.vbs
```

## 8. 恢复普通启动方式

不需要卸载任何代理软件。直接使用 Windows 开始菜单或原始官方快捷方式启动 Antigravity，即可恢复普通启动方式。

普通启动不会读取本项目专用的进程环境变量。

## 9. 卸载

1. 删除桌面的 `Antigravity (Proxy).lnk`。
2. 删除本项目目录（如果不再需要）。
3. 不要删除 Antigravity 官方安装目录。

本项目没有服务、计划任务、开机启动项或系统级环境变量，因此没有额外的后台组件需要清理。

## 10. Antigravity 更新后的重新验证

Antigravity 更新后，官方安装路径、官方快捷方式行为或子进程代理继承方式可能变化。建议：

1. 确认 `proxy-config.psd1` 中的代理监听仍在工作；
2. 重新运行桌面专用快捷方式；
3. 检查 Antigravity UI、登录、模型和 Agent 是否正常；
4. 如果更新后失败，再检查 Language Server 是否仍继承代理环境变量。

不要因为一次更新失败就开启整机 Global Mode 或 TUN；先确认具体子进程的代理行为。

## 11. 已知限制与常见问题

### 启动后仍然无法连接

- 确认本地代理监听器正在运行；
- 确认 `ProxyUrl` 使用的是 `http://`，而不是纯 SOCKS5 地址；
- 确认端口没有被其他程序占用；
- 确认 Antigravity 官方安装路径是默认路径；
- 确认目标组件确实遵循 `HTTP_PROXY` / `HTTPS_PROXY`。

### 为什么界面能打开，但 Agent 仍失败？

Electron 界面、Language Server 和 Agent 可能是不同进程。界面使用 WinINET 或自己的网络设置，并不代表子进程自动继承代理。这个项目专门把环境变量放在官方启动链的父进程环境中。

### 这个项目会代理所有流量吗？

不会。它只影响从该启动链继承环境变量、并且遵循这些变量的进程和网络库。忽略环境变量的组件、独立服务、其他软件或系统服务不在保证范围内。

### 会不会泄露代理密码？

示例配置不包含认证信息。不要把带账号密码的 URL、Token、Cookie、日志或个人路径提交到公开仓库。

### 这是地区限制绕过工具吗？

不是。它只是一个进程级 HTTP 代理启动器，不解除地区限制、不改变账号资格、不保证服务可用性。

## 12. 验证范围

公开项目的实现方式来自作者个人 Windows 环境中的实际使用方案。作者个人环境曾验证：Antigravity GUI、Language Server、bundled Agent 通过本地 HTTP 代理工作，并完成过真实模型请求。

公开通用版本未经独立跨设备测试。不同 Antigravity 版本、安装路径、代理客户端和网络库的行为可能不同。

## License

本项目采用 MIT License。详见 [LICENSE](LICENSE)。

