# Tailscale 与网络

## 尾网现状

- 尾网有 17 台设备。台式机 100.109.57.109，云电脑 100.123.171.106，本机 VM 100.117.32.115。

- 本机 VM 的 Tailscale 是受限代理包装，不支持 netcheck 和 ping，没有入站连接，流量走中继。

- 台式机和云电脑之间完全走中继，没有直连，延迟 300 毫秒到 1.3 秒。两台机器都在中国，这个延迟不正常。

## 测量方法

- 在台式机和云电脑上跑 `tailscale status`、`tailscale ping`、`tailscale netcheck`，用 WinRM 执行。

- 看三点。MappingVariesByDestIP 为真表示对称 NAT，打洞失败。PortMapping 为空表示没拿到 UPnP 或 NAT-PMP 映射。看两边实际走的中继是哪一个。

- 实测结果。台式机是简单 NAT，MappingVariesByDestIP 为假，公网 IPv4 是 27.44.63.7，UDP 通，PortMapping 为空。云电脑是对称 NAT，MappingVariesByDestIP 为真，公网 IPv4 是 182.42.224.13，没有 IPv6。最近中继是香港，台式机 156 毫秒，但两边选中继不一致，一边德国一边悉尼。控制端正常。

## 不要迁 Headscale

- 控制端只管身份，不管流量，迁 Headscale 要全部 17 台设备重新加入，得不到任何好处。

- 问题在 NAT 和中继选择，不在控制端。

## 修复顺序

- 先试免费办法。路由器开 UPnP，或者把 UDP 41641 转发到台式机 192.168.1.7，让台式机能被直连，延迟有望到几十毫秒。

- 再在香港搭自有 DERP 中继，约 340 毫秒，而且两边一致。官方中继留作备用，OmitDefaultRegions 保持关。

- 不要把 DERP 放到现有服务器上。Oracle 在日本，约 550 毫秒。GCP 在美国西部，约 375 毫秒。都比香港差。国内放一台中继能到几十毫秒，但要处理备案和端口。

## 路由器与宽带

- 家里路由器是华为 V175 FTTR 主网关，光纤，192.168.1.x，网关 192.168.1.1，Wi-Fi 是 H 和 H-5G，12 台设备在线。

- 宽带是广东东莞联通，公网 IP 27.44.63.7，AS17816，不是电信。公网 IP 是真实的，不是 CGNAT，端口转发能生效。

- 超管账号是 CUAdmin。2026-11-18 试过 5 组默认超密全失败，账号错 3 次触发锁定，锁 24 秒。广东联通很可能已通过 TR069 把超密改成随机的。

- SSDP 探测没有 UPnP IGD 回应，UPnP 是关着的。华为路由器的开关在更多功能、网络设置、UPnP，默认关闭。

- 不要再猜密码。等用户找装维要到 CUAdmin 超密后，登录一次，开 UPnP，或者加 UDP 41641 转发到台式机。然后在台式机上验证，`tailscale netcheck` 里 PortMapping 应该出现，`tailscale ping` 台式机和云电脑之间应该变成直连。

- 路由器设置没动过，不要复位。用户已表态，先试，找装维的话他去联系。

## 节点能不能跑在路由器里

- V175 是运营商锁定光猫，装不了软件，节点跑不进去。

- 要实现节点在路由器里跑、不用电脑，得换个能跑 Tailscale 的路由器，比如 GL.iNet，接在 V175 后面。或者把 V175 改桥接，用新路由当主路由，改桥接也要超密。

## Linux 服务器

- gcp-2 和 oracle-1 的 SSH 密钥全被拒，连不上，等用户提供正确私钥。

- 这也卡住了 DERP 部署，那两台服务器现在用不上。

## Available connectors

- [Tailscale](/opt/hatch/connectors/tailscale/CONNECTOR.md) for joining this VM to the tailnet and reaching private TCP services through the built-in proxy.

- Use `muse.connector_search` to find other connectors and `muse.skill_search` to find related skills.

- If nothing here fits, look on the web for a maintained API or MCP server and set it up with [Custom Connectors and MCPs](/opt/muse/skills/custom-connector-setup/SKILL.md), or build a custom connector.

## Alternatives for better results

- 直连争不到时，先接受香港中继的一致性，不要反复换中继地区。

- 路由器密码拿不到就不要在登录上磨时间，先把能做的中继方案准备好。
