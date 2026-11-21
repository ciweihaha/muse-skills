# UnBox 源列表修法

UnBox 点播源列表问题的一次完整修法：切源后列表变空、切不回来。仓库 `ciweihaha/UnBox`，分支 master，本地克隆 `~/workspace/unbox`。

## 现象（用户原话，2026-11-20 23:49 / 2026-11-21 07:49）

- 源列表要只显示支持的格式，不支持的不要显示。
- 切换列表后很难切换回来。

实机复现（云电脑，v0.9.17 测试构建）：设置里显示“当前来源：Cloudflare · **0 个站点**”，最后拉取 2026/11/21 01:00:17；点播页空白，无分类/线路下拉框。CF 下拉框有 3 个源（聚合万能源 44站合一·支持4K、OrionTV 聚合 38站、亚洲点播）；选“聚合万能源”显示已切换，但站点数仍 0、时间戳不变；重启后又回到“OrionTV 聚合（38站）”。

## 根因（两条，都有实测证据）

1. **`agg` 地址返回的不是 TVBox 配置，是 CMS 列表响应**：`https://ysp.ciweihaha.win/agg` 返回的 JSON 有 class/list、没有 sites；Go 实测 agg=0 站点，vod.json=1 站点（CMS，可用），oktv-asia.json=35 站点（全 CMS 但全是成人站）。`ImportVodSource` 把 0 站点当成功导入，**清空正在用的好源**，还把坏源记成当前源——这就是“切过去变空、切不回来”的机制。
2. **云电脑到 Cloudflare 的 IPv6 不通，且 Go 在 Windows 上不读 hosts**：curl 默认/-6 → 000，-4 → 200；hosts 里钉 104.21.23.111 只对 curl 有效，Go 用自己的解析器，app 根本拉不到源。修法：云电脑物理网卡 InterfaceIndex 2（Red Hat VirtIO Ethernet Adapter #2）禁用 IPv6 绑定，Tailscale 网卡（InterfaceIndex 14）不动。恢复命令：`Get-NetAdapter -InterfaceIndex 2 | Enable-NetAdapterBinding -ComponentID ms_tcpip6`。禁后手动推票 ok=3，取票服务不受影响。**笔记本不要禁 IPv6**（用户自己的机器，未授权；笔记本到 Cloudflare 的 IPv6 同样不通，但 CF 目录仍能拉到）。

## 修法（commit c60f787，2026-11-21，8 文件 +267/−11）

已发 v0.9.17（tag → c60f7877a5eefcedf1a7ee51b2d564b98ccd910f，Release https://github.com/ciweihaha/UnBox/releases/tag/v0.9.17）。

1. `ImportVodSource`：0 站点 → 报错“该源没有可用的点播站点”，绝不动正在用的源（`internal/shell/vodsource.go`）。
2. `FetchCfVodDirectory`：拉目录时并行验证各源的支持站点数（`config.CountSupportedSites`，与 `collectVodSites` 同一套判定：cms/http/js/jar，5 秒超时）；确认 0 站点的源从下拉框剔除；拉不到/解析不了的保留为未知（Sites=-1），不误删。
3. 前端 `resolveCfVodSources` 同样过滤 `sites===0`；正在用的地址即使被过滤，也仍显示为“正在用的源”（`frontend/src/vodSource.ts`、`App.vue`）。
4. 导入/刷新失败时下拉框拨回真实在用源（`loadVodSourceStatus()`）。
- 分类判定在 `internal/config/classify.go`（`Classify`：cms/js/http/已知 jar 引擎 = 支持；xpath/python/未知 jar = 不支持）；VOD 线路下拉框在 `frontend/src/App.vue:2457`；后端聚合在 `internal/shell/service.go:794`（`collectVodSites`）。
- 测试随修法一起进：`vodsource_test.go`（0 站点拒绝、目录过滤）、`supported_sites_test.go`、`vodSource.test.ts`。

## 验证方法（笔记本直验，2026-11-21，Lenovo 81TL）

测试构建 `C:\temp\unbox0917\UnBox-0.9.17-test.exe`（27,840,000 字节，SHA-256 `C4CF2AC76569E991AD805E08B64A9E56468DC336817896EC9E04869C752382ED`）。没用被拦的启动器、没跑 32 项 E2E；新写聚焦脚本直验。脚本在 `~/workspace/Verify-UnBox-SourceList.ps1`、`Probe-CfDropdown.ps1`、`Probe-SwitchBack.ps1`、`Probe-SwitchBack2.ps1`，笔记本上在 `C:\build\unbox-verify\`。记录在 `~/workspace/automation/runs/2026-11-20-unbox-0917-cloud.md`（“笔记本直验”一节）。

- **V1 下拉框过滤 0 站点源**：选项只有 `聚合万能源(44站合一·支持4K)`、`手动输入…`、`亚洲点播`；OrionTV 聚合（agg，0 站点）不在里面（3 次确认）。
- **V2 切源去再切回来**：切到亚洲点播站点数 1→35；切回聚合万能源 35→1，`上次拉取`更新。**判定标准：状态行从不含源名，切没切成要看站点数**（1=聚合万能源/vod.json，35=亚洲点播/oktv-asia），按源名判是错的。
- **V3 0 站点导入被拒、好源不被清空**（3 轮）：往点播源地址栏填 `https://ysp.ciweihaha.win/agg` 点导入，弹“该源没有可用的点播站点”，在用源站点数不变。
- **V4 列表永不为空**：首页无“暂无内容”，有内容。
- **S1 搜索**：聚合万能源源下搜“荒野求生”出结果（源停在亚洲点播时搜不到是源的问题，成人源里没有这片）。
- **S2 直播切台**：点 CCTV-1，状态“正在播放”。

CI 结论（run 44862830344，c60f787）：Go job success（shell 包新测试全绿，含 0 站点拒绝与目录过滤）；frontend job failure——只败在 Test 步骤，Build（含 vue-tsc）成功；与改动前完全相同的失败模式，本地 vitest 425/426，唯一失败是 master 上已有的 `layoutContracts.test.ts` 类似内容卡片宽度契约（有意不放宽），不是本次改动引入。

## 测试脚本经验（UIA，测 UnBox 时必读）

- 下拉框每次点选项前必须**重新打开**（选项只在打开时才在 UIA 树里），打开后**不能**按 ESC 再点；触发器不支持 UIA Invoke，得鼠标点。
- `Get-WindowRect` 只在 E2E 脚本里有，`UnBox.Uia.ps1` 里没有——脚本里调它会直接把整个函数打断；改用 `Get-ElementRect` + try/catch 包住滚动。
- 切源后不要立刻判，等下拉框/状态稳定；站点数是唯一可靠的切换判据。

## 已知限制

- **32 项完整 E2E 第 10 轮没跑成**：启动器（`unbox-r10-start.ps1` / `unbox-r10-run.ps1`，下载脚本 + 建计划任务执行）被系统安全检查拦下四次，用户四次点头仍拦。**不要读、不要跑这两个文件**，不要再试启动器模式。源列表修法靠上面 V1–V4 + S1/S2 直验，未走完整 E2E。
- 云电脑 IPv6 已禁（见根因 2），笔记本没禁。
- Windows 上传新 exe 前先停旧进程、拷完校验大小/SHA，失败写 `.fail`：2026-11-21 构建日志写了 BUILD DONE，但测试 exe 仍是旧版（旧进程锁着文件，Copy-Item 在 Continue 模式下静默失败，日志里没留报错）。
- 上游 v1.0.0 同步在分支 `sync/upstream-v1.0.0`（merge commit b1e93cbb），0.9.17 发完之后再做，不合并进本次发版。

## 发版（UnBox 的规矩）

- 版本走 tag：`v*` 标签 push 触发 `.github/workflows/release.yml`，五平台原生编译（Linux amd64/arm64、Windows amd64/arm64、macOS universal）+ 自动创建 Release 挂产物；不要用 API 传包（会 401）。
- GitHub 写操作只走经典 PAT（GitHub App 写操作 403、无 Workflows 权限）。Helper 在 `~/workspace/fongmi-tv-aar-inspect/`，一律 `python3 <路径>`：`github_create_tag.py <owner> <repo> <tag> <完整40位SHA>`；`github_push_batch.py <owner> <repo> <branch> <message> <files...>` 在仓库根目录跑，相对路径，提交信息里不要带提交哈希。
- 发版前先在真机上实测（先测后发），测试脚本留下来复用；修 bug 先确认问题再修。
