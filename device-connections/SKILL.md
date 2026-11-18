# 连接台式机、云电脑和安卓平板

## 先选对机器

- 用户的 Windows 应用先在云电脑上测，云电脑解决不了再用台式机。这是用户定的规矩。

- 安卓应用只能走台式机上的 adb 连平板测。云电脑上不装模拟器。

- 三台机器当前装了什么版本，先查 `~/workspace/automation/state.json`，那里是准的。

## 连台式机

- 台式机是 DESKTOP-MI11M34，Windows 11 Canary 26H2 Build 26300.9974 企业版，i5-10210U，16 GB 内存，MX230 显卡，只有 Wi-Fi，没有有线网卡。机器从不休眠，开机自动登录。

- Tailscale 地址是 100.109.57.109，局域网地址是 192.168.1.7，走 Wi-Fi H-5G。

- 用 WinRM 连，端口 5985，用户是 jiaheng。密码只走 `WINRM_PASS` 环境变量，绝不写进文件、聊天记录或技能里。

- 工具是 `~/workspace/.venv-winrm`（pywinrm）和 `~/workspace/dt.py`。命令是 `WINRM_PASS=<密码> ~/workspace/.venv-winrm/bin/python ~/workspace/dt.py desktop "<PowerShell 命令>"`。

- `dt.py` 会自动把运行环境的代理端口换成 3130 再连，不用手动改代理。

- 单条 WinRM 命令约 3000 字符上限。超长就把命令写进 `.ps1` 文件，用 `dt.py desktop -f <文件>` 跑，或者把脚本 base64 编码后在远端解码执行。

- 在 bash 双引号里写 PowerShell 命令时，把 `$` 转义成 `\$`，或者整段用单引号包起来。GUID 要加引号，含逗号的 BCD 值要用双引号包起来。

- 用 Python 生成 base64 的 PowerShell 脚本时，Windows 路径要用原始字符串，`\a` 会被当成响铃字符。

- 不要重启台式机。真要重启先问用户。

- 台式机 `D:\apks` 放着要装到平板上的 APK。`D:\win11-stable` 是重装准备，不要动。

## 连云电脑

- 云电脑是 nm6b2-ydn-8c16g，天翼云电脑，Windows Server 2022，Xeon 8 核，16 GB 内存，没有显卡，从不休眠。Tailscale 地址是 100.123.171.106。

- 用法和台式机一样，只是把 desktop 换成 cloud，用户是 Administrator，密码同样走 `WINRM_PASS` 环境变量。

- 云电脑只测 Windows 应用，不装模拟器。用户说过两次。

- 云电脑直连 GitHub 很慢，下载用 ghfast.top 镜像。HuggingFace 用 hf-mirror。

- 云电脑在对称 NAT 后面，Tailscale 打洞失败，和台式机之间只能走中继，延迟 300 毫秒到 1.3 秒。这是正常的，不要迁 Headscale。

- 云电脑上跑着 Ollama，地址是 127.0.0.1:11434，模型是 qwen2.5:1.5b，供 UnBox 字幕翻译用。OllamaServe 计划任务在登录时启动，会保持它运行。

## 连安卓平板

- 平板是华为 MediaPad M3 BTV-W09，Android 7.0 和 EMUI 5.0.4，麒麟 950，4 GB 内存，32 GB 存储，1600×2560。没有 root。

- 平板只能通过台式机上的 adb 连。adb 在 `%LOCALAPPDATA%\Android\Sdk\platform-tools\adb.exe`。先用 `dt.py` 在台式机上跑 `adb devices`，确认能看到序列号 28D6R17A17000103。

- 平板 Wi-Fi 地址是 192.168.1.23，和台式机在同一个 192.168.1.x 局域网里，AP 隔离是关着的。

- 装应用用 `adb install <apk>`。只有 32 位的包要加 `--abi armeabi-v7a`，否则会因为缺 libgojni.so 崩溃。

- 平板通过台式机代理上外网。台式机上 v2rayN 和 xray 监听 127.0.0.1:10808，端口转发 0.0.0.0:10809 到 127.0.0.1:10808，平板全局代理设成 192.168.1.7:10809。代理目前是开着的。

- 平板上装了 Tailscale 1.104.5（第三方重编版），但还没登录，用户说晚点再弄。不要自己去登。

- 测完把平板屏幕超时恢复到 120000，关闭常亮。

- 平板是共用的。动手前先确认有没有别的 agent 在用它，不要抢前台，不要强行停止别的包。当前谁在用、装了什么版本，查 `~/workspace/automation/state.json`。

## 整体拓扑

```
Tailscale 尾网
  这台 VM 100.117.32.115
  台式机 100.109.57.109
  云电脑 100.123.171.106（对称 NAT，只走中继）
  Oracle 和 GCP 服务器

家里局域网 192.168.1.x（华为 V175，网关 192.168.1.1）
  台式机 192.168.1.7 ── USB adb ──▶ 平板 192.168.1.23
  台式机 xray 10808 ── portproxy 10809 ──▶ 平板全局代理上网
  IPTV 盒子接光猫第二个口（单独 IPTV 口，不在 192.168.1.x）
```

## 常见问题

- 隧道断了（ProxyError、RemoteDisconnected、ReadTimeout）就等一等再试，过去都是自己恢复。不要连续猛试。定时任务每 10 分钟探测一次，通了会报回来。

- 每次调用代理口令都会换。不要用上一次 shell 里的旧口令。`dt.py` 每次都从 `$HTTPS_PROXY` 现取。

- 本机不能重启。reboot 会报连不上总线，不要再试。

- WinRM 连不上先确认 `WINRM_PASS` 在同一个 shell 里设好了，再确认目标机器的 Tailscale 在线。桌面和云电脑都从不休眠，一般不是机器睡了。

- adb 里看不到平板先检查 USB 线和台式机上的 adb 服务，再确认平板亮着屏且 USB 调试开着。不要在别的 agent 测试中途拔线。

- 平板上不了网先查台式机上的 xray 是否在跑（127.0.0.1:10808），再查端口转发和防火墙规则 Tablet Proxy 10809，最后查平板的全局代理设置。

- 台式机和云电脑之间 Tailscale 一直走中继、没有直连是正常的。云电脑是对称 NAT。想争直连要开路由器 UPnP 或把 UDP 41641 转发到台式机 192.168.1.7，这需要联通超管 CUAdmin 密码，找装维要。不要猜密码，账号错 3 次会锁 24 秒。

- 在这台 VM 上不要跑 `tailscale ping` 或 `tailscale netcheck`，这里的 Tailscale 是受限的代理包装，不支持这些命令。需要在台式机或云电脑上跑。

## Available connectors

- [Tailscale](/opt/hatch/connectors/tailscale/CONNECTOR.md) for joining this VM to the user's tailnet and reaching private TCP services through the built-in proxy.

- Use `muse.connector_search` to find other connectors and `muse.skill_search` to find related skills.

- If nothing here fits, look on the web for a maintained API or MCP server and set it up with [Custom Connectors and MCPs](/opt/muse/skills/custom-connector-setup/SKILL.md), or build a custom connector.
