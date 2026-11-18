# 台式机清理维护

## 原则

- 只卸载确认没用、且卸载后不影响现有工作的软件。拿不准的一个都不卸，列出来让用户自己决定。

- 操作前确认目标，卸载后验证一遍。只做安全、可逆的操作。

- 不要关 Hyper-V，不要改系统设置，不要删用户文件，不要动 `D:\win11-stable`。

## WSA 与模拟器

- WSA 社区版 2407.40000.4.0 已卸。卸载用 `Remove-AppxPackage`，必须用完整包名 `MicrosoftCorporationII.WindowsSubsystemForAndroid_2407.40000.4.0_x64__8wekyb3d8bbwe`，用包族名会报 `0x80073CFA`。

- 验证三点。`Get-AppxPackage` 查不到 WSA，数量是 0。`WsaService` 服务已消失。`%LOCALAPPDATA%\Packages` 下无 WSA 残留文件夹。

- `D:\wsa` 整个文件夹已删，约 3.73 GB，包括解压出的 WSA 目录 1.85 GB、官方版安装包 `official.msixbundle` 1.47 GB、`WSA_NoGApps.7z` 0.53 GB。删完 D 盘剩约 217 GB。以后想重装 WSA 要重新下载安装包。

- WSA 本来就没法用。官方版和社区版都装得上、服务都跑得起来，但虚拟机起不来，根因是 Win11 Canary 和 Hyper-V 不兼容。

- 模拟器 2026-11-17 已删，这次确认干净。检查位置是 Program Files、服务、注册表卸载项，找 BlueStacks、夜神、雷电、MEmu、MuMu 的痕迹。

## 安卓开发工具

- Android Studio IDE 已卸，静默卸载加 `/S`，释放约 3.32 GB。

- SDK 和 adb 留着，平板测试要用。SDK 在 `%LOCALAPPDATA%\Android\Sdk`，641 MB，卸载前后大小一致。

- 卸载前后各验证一次。`adb version` 正常，`adb devices` 显示平板序列号 `28D6R17A17000103 device`。平板全程不能断开。

- 如果卸载 Android Studio 时提示要连 SDK 一起删，或者卸完发现 adb 没了，立刻停手，先把独立版 platform-tools 装回来并验证能连平板，再继续。

## 绝对不要动的东西

- Tailscale、Chrome、v2rayN 和 xray、TiviMateWin、Voxxwire、Android SDK 和 adb、Node 和 npm、Chrome DevTools MCP、Hyper-V、`D:\win11-stable`、`D:\apks`。

- v2rayN 和 xray 是平板代理要走的，Tailscale 是连机器要走的，adb 是平板测试要用的，动了任何一个都会断测试链路。

## 拿不准、没卸的清单

- 这些软件用途不确定，一个都没动，包括 AdsPower、Cloudflare WARP、天翼云电脑客户端及 UsbDk、TeleAgent、WorkBuddy、夸克浏览器、微信和企业微信、QQGame 和 FunPlus、大白菜、`D:\` 下的旧安装包杂物文件夹、`%APPDATA%\Goldberg SteamEmu Saves`。

- `%APPDATA%\Goldberg SteamEmu Saves` 是 Steam 游戏存档相关，不是安卓模拟器，不要误删。

## 记录

- 写一份记录到 `~/workspace/automation/runs/<日期>-desktop-cleanup.md`，包括卸了什么、释放了多少空间、留了什么及原因、拿不准没卸的清单。

## Available connectors

- Use `muse.connector_search` to find other connectors and `muse.skill_search` to find related skills.

- If nothing here fits, look on the web for a maintained API or MCP server and set it up with [Custom Connectors and MCPs](/opt/muse/skills/custom-connector-setup/SKILL.md), or build a custom connector.

## Alternatives for better results

- 卸载前先看这个软件有没有正在跑的服务或定时任务，有就先停掉再卸。

- 大文件夹删之前先确认里面没有用户自己的文件，只删安装包和缓存。
