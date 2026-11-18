# 直播源与凤凰秀

## 服务器地址

- 影视 app 点播配置是 `https://live.ciweihaha.win/vod.json`，导这个配置，20 站合一，接口是 `https://live.ciweihaha.win/vod-api/`。

- 影视 app 直播是 `https://live.ciweihaha.win/live.m3u`，56 个台。

- OrionTV 点播聚合地址是 `https://ysp.ciweihaha.win/agg`，写死在代码里，37 站。

- M3U 在腾讯云函数里维护，函数名 `ysp-ticket`，Node 18.15，ap-guangzhou。凤凰三台走函数里的 `/fh/fhzx`、`/fh/fhzixun`、`/fh/fhhk` 三条路由。

## 凤凰秀官方源

- API 是 `https://api.fengshows.cn/`，请求头必须带 `fengshows-client: app(fs-web,1000000);`，有 token 时再带 `token:` 请求头，值是裸 JWT。

- 每次播放都要现取签名，分两步。先请求 `GET /hub/resource/live/{channel_id}?platform=web`，再请求 `GET /hub/live/auth-url?live_qa={fhd|hd}&live_id={channel_id}`，取 `data.live_url`，是 dispatch.fengshows.cn 8484 端口的签名 HTTP-FLV。

- 三台 channel_id 写死。凤凰中文是 `f7f48462-9b13-485b-8101-7b54716411ec`，凤凰资讯是 `7c96b084-60e1-40a9-89c5-682b994fb680`，凤凰香港是 `15e02d92-1698-416c-af2f-3e9a872b4d78`。

- 签名几分钟就过期，云函数必须每次点播现取签名，绝不能把签名 URL 写死进 M3U。

- fhd 是 720p，不是 1080p。uhd 是 640x360。会员号目前最高就 720p。

## Token 获取

- Token 是登录后 Cookie 里的 `App.user.token`，不是 HttpOnly。登录页是 `https://www.fengshows.com/live`，手机号加密码登录，区号默认加 852，要切到加 86。API 返回 code 10005 表示 token 过期，登出会让 token 失效。

- Token 有效期 30 天。当前 token 到 2026-12-18 过期，账号名是真正的人工。注意 token 里 vip 字段是 0，和用户说的付费会员对不上，1080p 没拿到。

- 拿 token 用 Chrome DevTools MCP 在本机登录取。云电脑被 fengshows.com 拦了 HTTP 418，本机能开。MCP 配置在 `~/workspace/mcp-token/`，driver.js 和 relay.js 可复用。

- MCP 的坑都已踩过。Chrome 是 `/opt/meta-chromium/chrome`，要加 WebGL 允许域名参数和 `--no-sandbox`，`--chromeArg` 必须用等号形式。Chrome 的 `--proxy-server` 不认带口令的代理地址，要走本地 relay 127.0.0.1:18080 加 Proxy-Authorization。出口做 TLS 中间人，MCP 要加 `--acceptInsecureCerts`。登录表单的手机号输入框不在无障碍快照里，先用 evaluate_script 聚焦再输入。登录请求可能要等约 90 秒。

- Token 只走环境变量和请求头，绝不写进任何文件。云函数里用 `FENGSHOWS_TOKEN` 环境变量。

- 腾讯云密钥也只走环境变量，绝不写进文件。用完即弃，下次要更新云函数时再找用户重新提供。

## 云函数更新

- 云函数已在做官方流程，fhResolve 走 auth-url 拿签名再 302 到 dispatch。更新时只换 `FENGSHOWS_TOKEN` 环境变量，不要动 M3U。

- 不要跟随 dispatch 的 301 跳到 qitv，直接 302 到 dispatch 原地址。当前版本是 v4.7.13。

- 不要切到 hd，hd 和 fhd 走同一条链，dispatch 301 到 qitv，只是画质从 720p 降到 480p，没有任何好处。

- 不要做云函数代理模式。SCF 超时上限 900 秒，代理流单次最多撑 15 分钟，播放器仍会断一次，只加复杂度和成本。

## 验证标准

- 测流是否断，用 curl，而且不要加 `--retry`。curl 默认不重连，断流会立即以 exit 18 退出，35 秒跑满会被 `--max-time` 掐断报 exit 28。每 5 秒采样一次字节数，字节数一直在涨才是同一条连接连续在跑。

- ffmpeg 的墙上时间不能当证据，ffmpeg 断了会自动重连，20 秒墙上时间不等于一条连接撑了 20 秒。

- 平板实测用数字键直输台号，51、52、53 是凤凰三台，50 是深圳卫视对照，经台标核对无误再开始计时。频道顺序和台号对不上，不要用上下键换台。标准是连续播 30 秒，对照台正常。

- 已知矛盾如实记录。curl 三条路、6 次 35 秒全部连续，平板三台 30 秒内必报 CUTOFF(10004)，对照正常。两种可能，一是 qitv 断流偶发，二是断流只发生在 ExoPlayer 或平板路径上，数据区分不了。

- 结论是源侧量不到断流时不要再改源，解法移到 mytv 应用侧做直播断线自动重连。

## Available connectors

- [Github Pat](~/workspace/custom-connectors/custom.github-pat/CONNECTOR.md) for reading the SCF function code if it lives in a repo.

- Use `muse.connector_search` to find other connectors and `muse.skill_search` to find related skills.

- If nothing here fits, look on the web for a maintained API or MCP server and set it up with [Custom Connectors and MCPs](/opt/muse/skills/custom-connector-setup/SKILL.md), or build a custom connector.

## Alternatives for better results

- 源侧改不动时优先改播放器侧的容错，不要反复折腾云函数。

- 每次改完云函数先在 curl 上验 35 秒，再上平板，不要直接拿平板当第一道验证。
