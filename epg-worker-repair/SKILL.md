# 节目单 EPG 缺台修复

`ysp.ciweihaha.win/epg.xml` 缺台的诊断、手工修补、以及 ysp-ticket Worker 补抓的彻底修复。2026-11-21 实战沉淀：早上合并央视频接口报 530 丢了 cctv1/cctv4/cctv5，14:00 补抓撞上免费版子请求上限没补上；手工补回 91 个节目，彻底修法是把补抓改到预热前面（部署 2ea0dc77）。

## 架构：节目单从哪来

- 线上文件 `https://ysp.ciweihaha.win/epg.xml` 是 **Cloudflare KV 里的静态文件**：namespace `iptv-playlists`（`71548caee2184029bc46dea9884f92eb`），key 就叫 `epg.xml`。`ysp-ticket` Worker（`ysp.ciweihaha.win` 是它的 custom domain）原样返回，不实时生成。
- 生成链路全在 ysp-ticket Worker 内：频道表按 2 等分抓成 shard（`epg-merge:shard1/2:<YYYYMMDD>`，2 天 TTL）→ 合并写 `epg.xml` + `ysp.m3u`；单频道数据来自央视频官方节目单接口（见“手工修补”）。
- 定时（ysp-ticket 的 3 个 cron，都是 UTC 时间）：
  - `32 21 * * *`（北京时间 05:32）：全量合并分片 A；`47 21 * * *`（05:47）：分片 B + 合并写文件。
  - `0 */3 * * *`（每 3 小时）：取票预热 `yspWarmup()` + 补抓 `epgMergeTopup()`（北京时间 06:00 之后才跑补抓；缺失为 0 时只写 status，不抓取）。
- 合并保护：频道数 < 50 时**拒绝覆盖** `epg.xml`（保留旧文件）；每次合并写状态 `epg-merge:status:<YYYYMMDD>`（7 天 TTL），字段有 `ok / channels / programmes / missing / source / ts`。
- 单频道缓存：EPG_CACHE_MS 6 小时；抓取失败/空结果负缓存 10 分钟（EPG_NEGATIVE_CACHE_MS）。
- **不要“修” `live.ciweihaha.win/epg.xml` 的 404**：那是用户 2026-10-07 亲口要求下线的，iptv-live 代码里明确返回 404。应用读的是 `ysp.ciweihaha.win/epg.xml`（live.m3u 首行 `x-tvg-url` 指的就是它）。

## 1. 查哪几个台缺节目单

live.m3u 里每个台有 `tvg-id`，epg.xml 里每个台有 `<channel id>`；播放器按这个 id 配对，id 对不上或文件里没有 = 没节目单。

```bash
curl -s -m 20 -o /tmp/live.m3u "https://live.ciweihaha.win/live.m3u"
curl -s -m 20 -o /tmp/epg.xml "https://ysp.ciweihaha.win/epg.xml"
python3 - <<'EOF'
import re
m3u = open('/tmp/live.m3u', encoding='utf-8').read()
epg = open('/tmp/epg.xml', encoding='utf-8').read()
epg_ids = set(re.findall(r'<channel id="([^"]+)"', epg))
print("m3u 台数:", len(re.findall(r'#EXTINF:-1', m3u)), "epg 台数:", len(epg_ids))
for line in re.findall(r'#EXTINF:-1[^\n]*', m3u):
    tid = (re.search(r'tvg-id="([^"]*)"', line) or [None, '(no tvg-id)']).group(1)
    name = (re.search(r',\s*(.*)$', line) or [None, '']).group(1).strip()
    if tid not in epg_ids:
        print(f"缺节目单: {tid} | {name}")
EOF
```

2026-11-21 实测缺 9 个：cctv1（CCTV综合）、cctv4（CCTV中文国际）、cctv5（CCTV体育）是有 tvg-id 但文件里没数据；凤凰中文/资讯/香港、香港卫视、澳门莲花、BBC 新闻是 live.m3u 里根本没写 tvg-id——后 6 个本来就没有节目单数据源（见“已知缺口”），不在缺台修复范围内。

## 2. 定位原因：读 status 和 shard 注释

Cloudflare helper：`python3 ~/workspace/custom-connectors/custom.cloudflare/bin/cf_api.py GET <路径>`（路径不带 `/client/v4`；GET 写 stdout；PUT 的第 3 个参数是 body 文件）。账号 `12c67a5c82b93a2ead595701c8493b5f`。

```bash
NS=71548caee2184029bc46dea9884f92eb
CF=~/workspace/custom-connectors/custom.cloudflare/bin/cf_api.py
python3 $CF GET /accounts/12c67a5c82b93a2ead595701c8493b5f/storage/kv/namespaces/$NS/values/epg-merge:status:20261121
python3 $CF GET /accounts/12c67a5c82b93a2ead595701c8493b5f/storage/kv/namespaces/$NS/values/epg-merge:shard1:20261121 | head -c 600
python3 $CF GET /accounts/12c67a5c82b93a2ead595701c8493b5f/storage/kv/namespaces/$NS/values/epg-merge:shard3:20261121 | head -c 600
```

status 的 `missing` 列出当天缺的台，`source` 说明是哪次写入（`morning` 全量合并 / `topup` 补抓）；每个 shard 开头的 HTML 注释有 `failed=<台>(<原因>)`。两种典型原因：

| 现象 | 含义 | 怎么办 |
|---|---|---|
| `EPG 上游 HTTP 530` | Worker 抓央视频接口时上游报错（Cloudflare 530 = 源站不可达），那次合并丢了这几个台 | 等补抓；补抓也失败时手工修补（§3） |
| `Too many subrequests by single Worker invocation` | 撞上免费版单次调用 50 个子请求上限 | 根因见 §4：补抓和预热挤在同一次调用里，预热先占满；修法是换顺序 |

2026-11-21 的证据链：status 写着 `missing=["cctv1","cctv4","cctv5"]`、`source=topup`；shard1 注释 `failed=cctv1/cctv3/cctv4/cctv5/cctv6/cctv7(EPG 上游 HTTP 530)`；shard3 注释 `failed=cctv1/cctv4/cctv5(Too many subrequests...)`。cctv3/6/7 在后续补抓里修好了，只剩 cctv1/4/5。

## 3. 手工修补：从源头接口直接拿数据写回 KV

适用：缺的台当天就得回来，等不及下次定时任务。只写 `epg.xml` 一个 KV key，不动 shard/status、ysp.m3u、live.m3u、Worker 代码。

1. **先备份线上文件**：下载 `https://ysp.ciweihaha.win/epg.xml` 存好，记下写前台数/节目数。
2. **调央视频接口拿缺台的数据**（沙箱直调是通的，HTTP 200；从 Worker 调可能 530——这正是缺台的原因）：
   `https://capi.yangshipin.cn/api/yspepg/program/<livePid>/<YYYYMMDD>`，请求头 `User-Agent: Mozilla/5.0`、`Referer: https://www.yangshipin.cn/`。日期是北京时间 `YYYYMMDD`。
   livePid 查 ysp-ticket 脚本里的 `YSP_CHANNEL_NAMES` / `YSP_BY_ID` 表；已知的三个：cctv1=`600001859`、cctv4=`600001814`、cctv5=`600001818`。
3. **解码 protobuf**：按 ysp-ticket 脚本里 `epgDecode` 的规则（varint + length-delimited 字段）：外层字段 2 = 一个节目条目；内层字段 2 = 标题（UTF-8 字符串）、字段 3 = 开始时间（秒）、字段 4 = 结束时间（秒）。核对：时间都在当天、标题是中文、无乱码。
   模板脚本：`~/workspace/automation/runs/2026-11-21-epg-manual-repair/repair-epg.py`（解码逻辑可直接复用；它引用的 `/tmp/epg-*.bin` 是临时文件，已不在，重新下载即可）。
4. **拼 XMLTV 片段**，格式照现有文件（display-name 照 Worker 里 `EPG_CHANNEL_NAMES` 表，如 `CCTV-1 综合` / `CCTV-4 中文国际` / `CCTV-5 体育`）：
   ```xml
   <channel id="cctv1"><display-name>CCTV-1 综合</display-name></channel>
     <programme channel="cctv1" start="20261121010200 +0800" stop="20261121013500 +0800">
       <title lang="zh">今日说法-2026-212</title>
     </programme>
   ```
   时间格式 `YYYYMMDDHHMMSS +0800`；`& < >` 要转义。把 3 个台的 `<channel>` 和全部 `<programme>` 插到 `</tv>` 之前；**原有内容一个字不改**。
5. **写回 KV**：`python3 cf_api.py PUT /accounts/12c67a5c82b93a2ead595701c8493b5f/storage/kv/namespaces/71548caee2184029bc46dea9884f92eb/values/epg.xml <修好的文件>`，返回 `{"success":true}`。
6. **写后核对**（缺一不可）：本地 XML 解析通过；台数 = 写前 + 补的台数、节目数 = 写前 + 补的节目数；KV 读回一致；**重新下载 `https://ysp.ciweihaha.win/epg.xml` 核对**（应与本地文件逐字节一致，或只差末尾换行）。

2026-11-21 实绩：补 cctv1 31 个、cctv4 38 个、cctv5 22 个节目（共 91 个），文件从 60 台 2294 个节目变成 63 台 2385 个；重新下载 md5 `9d03c9881569d6b3536de3f9393f4d6f`，线上已生效。记录在 `~/workspace/automation/runs/2026-11-21-epg-manual-repair.md`。

注意：下次定时全量合并（明早 05:32/05:47）会按正常流程重写 epg.xml；如果那时接口又 530、补抓又撞上限，缺的台可能再次丢失——手工修补不管这个，要靠 §4 的彻底修复。

## 4. 彻底修复：补抓改到预热前面

根因：`0 */3 * * *` 那次定时触发在同一次 Worker 调用里先跑取票预热（约 50 个子请求），再跑补抓；免费版上限 50，预热先占满，补抓直接报 `Too many subrequests`。补抓本身只用几个子请求。

改法（ysp-ticket 脚本 `scheduled` 处理函数里 `0 */3 * * *` 那个分支）：**先跑补抓、后跑预热**，两块各自的 `try/catch` 和 `bjHour >= 6` 条件原样保留，只换顺序；注释里说明为什么（2026-11-21 加的 3 行注释原文）：

```
// 补抓必须先跑：免费版单次调用最多 50 个子请求，补抓只用几个，预热约 50 个；
// 先跑补抓能拿到新鲜预算，预热是尽力而为、少预热几个票不影响（票可按需现取）。
// 反过来先预热会把预算占满，补抓报 Too many subrequests 失败。
```

没动：分片/合并逻辑、`EPG_MERGE_*` 常量、`CRON_A/B/C` 的值、fetch 路由、其他函数、定时任务。

### 部署要点（ysp-ticket，multipart module 上传）

部署脚本：`~/workspace/automation/runs/2026-11-21-epg-topup-reorder/deploy-ysp-ticket.py`（参照 `~/workspace/automation/vod-sources-refresh/merge.py` 的 `cf_put_module` 做法），`main_module=worker.js`，metadata 带 `compatibility_date: 2026-11-12` + `usage_model: standard`。

- **部署前先读 settings**（`GET /accounts/.../workers/scripts/ysp-ticket/settings`），记下全部 9 个绑定：3 个 KV（FENGSHOWS_KV `d23bd29a586c47c8b4d183569faf5ecb`、IPTV_PLAYLISTS `71548caee2184029bc46dea9884f92eb`、YSP_KV `265d3be54a154d1984921b8ca6f910e3`）+ 6 个 secret（FENGSHOWS_PASS、FENGSHOWS_USER、TOKEN_RELAY_KEY、WARMUP_KEY、YSP_CKEY_TEA_KEY、YSP_GUARD_TEA_KEY）。**部署前先备份线上脚本。**
- **metadata 只带 3 个 KV 绑定**：`secret_text` 声明时必须带值，不带值 Cloudflare 报 400 / 10021（`invalid or missing text property`）；secret 的值写后不可读、不能也不应去找。已验证的行为：**省略的 secret 绑定不会被清空，只有省略的 KV 绑定会被清空**（iptv-live 两次部署只带 KV 绑定后 secret 都在，3/3）。反过来，漏带一个 KV 绑定就会把它清空——2026-11-21 凌晨就是这么丢过 YSP_KV/FENGSHOWS_KV 导致取票 503 的。
- **部署后立即核 settings**：9/9 绑定全在、3 个 KV 的 namespace_id 不变、compatibility_date 不变。再验：`https://ysp.ciweihaha.win/epg.xml` 仍 200 且台数/节目数不变；拉回线上脚本，SHA 与部署文件一致，且补抓确实在预热前面。
- **验播放地址要用带 `.m3u8` 的**：`https://live.ciweihaha.win/ysp/cctv1.m3u8` 应 200。不带后缀的 `/ysp/cctv1` 固定 404，是 iptv-live 路由正则（`^\/ysp\/([a-z0-9]+)\.m3u8$`）的既有行为，不是部署造成的故障，别误判。

2026-11-21 实绩：部署 `2ea0dc77d94843d883cff60515362ee0`（16:33），改后脚本 SHA-256 `3dfbb1b6e9a2d4dae13ebfa97e2cd3eb8398b10533f807193178c78fe55d8bdb`；settings 9/9 一致；epg.xml 仍 200、63 台 2385 个节目（手工修好的文件原样 untouched）；`/ysp/cctv1.m3u8` 200、`cctv5.m3u8`（走 KV 票）200。记录在 `~/workspace/automation/runs/2026-11-21-epg-topup-reorder.md`。下一次补抓北京时间 17:00，是新顺序第一次运行——事后读 `epg-merge:status:<日期>`，`source=topup` 的 `missing` 应为空。

## 5. 约束清单

- **免费账号每账号最多 5 个 cron 触发器，已满**：ysp-ticket 3 个 + okys-agg 每 30 分钟 1 个（`*/30 * * * *`）+ uptimeflare 每 5 分钟 1 个（`*/5 * * * *`）。想加独立的补抓 cron（代码里预留的 `EPG_MERGE_CRON_C='30 4 * * *'`，即北京时间 12:30）得先腾位置；直接加报 10072。schedules 的 PUT body 是**裸 JSON 数组** `[{"cron": "..."}]`，包一层 `{"schedules": [...]}` 报 10026 解析失败。
- **不要动 okys-agg / uptimeflare 的 cron**：一个是点播聚合预热，一个是 uptime 监控（status.ciweihaha.win），都没确认可删。
- **6 个其他地区台无节目单数据源**：凤凰中文/资讯/香港、香港卫视、澳门莲花、BBC——央视频接口没有它们的数据，live.m3u 里也没 tvg-id；要补得先找各自的数据源 + 给 m3u 加 tvg-id，是另一个任务，不在本次修复范围内。
- **取票链路别碰**：补抓改顺序不影响取票；预热少跑几个票只是少预热，票按需现取 + KV 缓存 225 分钟兜底。
- Skill 里不写密码、token、密钥；KV namespace id 和公开接口地址可以写（上面都已写明）。
