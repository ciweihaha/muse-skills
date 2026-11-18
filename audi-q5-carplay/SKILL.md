# 奥迪 Q5 CarPlay 刷机

## 车况

- 2019 款奥迪 Q5，标准轴距（不是 Q5L），quattro 四驱。
- 教程仓库 `ciweihaha/Audi-Q5-CarPlay`（2026-11-09 建，README，提交 69ea346），覆盖 2018/2019 款 Q5 标准轴距。

## 关键前提

- 原车有线 CarPlay 必须已经能在中控主屏上正常用。这个改装是把 CarPlay 扩展到虚拟座舱（Virtual Cockpit）上，不是开通 CarPlay。原车没有有线 CarPlay 的话，这条路走不通，改用外接 CarPlay 盒子方案。

## 软件路线

- 机头是 MHI2Q，用 M.I.B. 工具 + `luka-dev/mib2q-carplay-rgi`。
- 步骤指南：`realshaunoneill/audi-mhi2q-carplay-cockpit`。
- 已验证的固件版本：美版 P5087、欧版 P5092 MU1329、国行 K1004、日版 P1017；固件从 mibvault.com 拿。
- 外接 CarPlay 盒子是零风险的替代方案，不动原车系统。

## 还缺的信息（等用户发）

- MMI 列车号（train）、MU 号、媒体驱动版本。用户 2026-11-08 已答应发，至今没发。拿到这三项才能确认用哪个固件和安装包。
- 没拿到之前不要让用户动手刷机，版本不对有变砖风险。
