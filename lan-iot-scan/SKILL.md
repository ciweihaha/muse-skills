# 局域网与物联网扫描

## 扫描方法

- 从台式机上用 WinRM 扫局域网 192.168.1.0/24。并行 ping 扫 192.168.1.1 到 254，跑两轮，再读 arp 表。

- 每个在线设备做 MAC 厂商查询、NetBIOS、SSDP、mDNS 发现，再轻量探测常用 TCP 端口，包括 80、443、8000、8080、6668、1883、554、9000、7000。

- 每个设备 ping 5 次，记延迟和丢包。

- 不要用太重的端口扫描，不要对不认识的设备乱发包。平板可能正被另一个 agent 用来测应用，不要动平板上的应用。

## 认设备

- 美的空调看 OUI，A0-68-1C 和 30-B2-37 都是美的。图微安创 TwoWing 模组 1C-5E-E6 是涂鸦生态的智能插座或开关类，具体是哪件要在手机 App 里确认。

- 智能电视看青岛精准电子 38-64-07，这个厂的 Wi-Fi 模组多用于智能电视和多媒体设备。

- 随机 MAC 是开了私有 Wi-Fi 地址的手机，iPhone 或安卓。ARP 可见但 ping 不回的是休眠中的设备。

- 第一轮在 ARP、第二轮消失的是已休眠或离线的设备。

- 2026-11-18 的实测结果。13 个设备，物联网 5 个，包括美的空调 3 台、涂鸦插座或开关 1 个、智能电视 1 台。路由器、台式机、平板、小米手机和几台私有地址手机都在。联通 IPTV 盒子扫不到。

## 健康检查

- 查三点。无 IP 冲突，无重复 MAC，无陌生或可疑设备。

- 延迟正常。空调 12 到 14 毫秒，智能家居设备 4 毫秒，电视 7 毫秒。物联网设备 5 次 ping 丢 1 次是射频正常波动，不算故障。

- 所有物联网设备常用端口全关，NetBIOS、SSDP、mDNS 全无回应，说明它们不暴露本地管理界面，都是云端管理。

## 优化的边界

- 这些物联网设备远程优化不了。美的空调走美居 App 连美的云，插座走涂鸦或米家 App，局域网里改不了它们。

- 能确认的只有在线状态和延迟正常。不要硬凑优化动作，没改就是没改，如实说。

- 不要改任何设备的系统设置，不要恢复出厂，不要断开任何设备，不要动 IPTV 盒子和光猫的运营商配置。

- 电视的 Wi-Fi 信号、空调连的是 2.4G 还是 5G，要进路由器后台或在设备上看。路由器现在进不去，等拿到 CUAdmin 超密后才能查。

- 空调定时、插座场景要在各自的手机 App 里设，局域网里设不了。

## IPTV 盒子扫不到的原因

- 联通 IPTV 盒子单独接在光猫下面第二个口，也就是 IPTV 口，专门看电视，不在 192.168.1.x 局域网里，所以扫不到。

- 它走单独的 IPTV 网络，直播是组播发给盒子的，和盒子本身绑在一起。就算把账号密码拿出来，电视也收不到那些台。

- 如果想看它，先确认它开着、插在哪根网线。不要从局域网里硬找，找也找不到。

## 记录

- 写一份记录到 `~/workspace/automation/runs/<日期>-lan-iot-scan.md`，包括设备清单、健康检查结果、改了什么、没改什么及原因、需要用户自己动手的事。

## Available connectors

- Use `muse.connector_search` to find other connectors and `muse.skill_search` to find related skills.

- If nothing here fits, look on the web for a maintained API or MCP server and set it up with [Custom Connectors and MCPs](/opt/muse/skills/custom-connector-setup/SKILL.md), or build a custom connector.

## Alternatives for better results

- 扫两轮再下结论，休眠设备第一轮在第二轮不在，单轮会漏。

- 设备认不出来时先写厂商和 IP，不要猜型号，让用户在手机 App 里对一下。
