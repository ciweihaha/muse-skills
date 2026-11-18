# TiviMateWin

Windows 直播播放器（WPF + LibVLC），带直播实时英翻中字幕。仓库 `ciweihaha/TiviMateWin`，分支 main，本地克隆 `~/workspace/tivimatewin`。

## 版本与发版

- 版本唯一事实源：`TiviMateWin.csproj` 的 `<Version>`，按 1.4.29 → 1.4.30 这样递增。
- GitHub 写操作只走经典 PAT（GitHub App 写操作 403、无 Workflows 权限；workflow 文件也要走 PAT）。Helper 在 `~/workspace/fongmi-tv-aar-inspect/`，一律 `python3 <路径>`：
  - `github_push_batch.py <owner> <repo> <branch> <message> <files...>`：推代码，在仓库根目录跑，文件用相对路径；提交信息里不要带提交哈希。
  - `github_create_tag.py`：打 tag，要完整 40 位 SHA；打 tag 触发 `push: tags` 的 Release 工作流出包。
  - Release 安装包走 API 上传会 401，用 tag 触发的 Release 工作流出包，不要用 API 传包。
- 推之前先 `git fetch` 核对本地与远程 HEAD 一致；有冲突先同步再推，绝不 force-push。
- 不要改 `LOOP.md` / `state.json`（循环调度用的）。
- `feature/live-subtitles` 分支已合并进 main，不要再碰那个分支。
- 最新版：v1.4.31（2026-11-18，提交 dd5086d7，Release https://github.com/ciweihaha/TiviMateWin/releases/tag/v1.4.31 ，`TiviMateWin-win-x64.zip` 160,870,399 字节，SHA-256 `272397e4a1ae21be31990bd55253ef960b4cc169c11bb0a224896acccabc550f`）。v1.4.31 桌面 24 项复测 12/24 通过（播放、换台含首次直接验证的鼠标双击、搜索、收藏、小窗、全屏、侧栏、源列表、刷新）；EPG 修复与重连降频已验证；关字幕崩溃因对话框阻塞未复测（见“已知遗留问题”）。

## 构建与测试

- .NET 11 SDK：台式机 `C:\dotnet11`（11.0.100）；本机 Linux `~/workspace/.dotnet11`（11.0.100）。Linux 上 restore Windows 目标需 `-p:EnableWindowsTargeting=true`。
- 本机只有 1.5 共享 CPU，测试跑不动（vstest 超时）；以台式机构建+测试与 CI 为准。
- 台式机构建在 `D:\build\TiviMateWin-main`（从 main zipball 解压）：`dotnet build -c Release`、`dotnet test`（v1.4.31 是 92/92 全过）。`dotnet test` 要用测试项目显式路径，否则跑不起来。
- `--selftest` 经 WinRM 挂起（非交互会话环境问题，不计为失败；CI 的自测是权威验证）。
- 2 个已有的 NU1510 警告：ProtectedData 包在 .NET 11 里自动可用，建议移除引用。

## 实时字幕管线（进程内，无 Ollama）

- 链路：音频抓取 → Whisper.NET（tiny.en）听写 → LLamaSharp（Qwen2.5-1.5B GGUF）翻译 → 画面叠字幕。模型首次打开时下载缓存（共约 1.26GB），不打进安装包。
- 关键文件：`Services/LiveSubtitleService.cs`、`LiveSubtitlePipeline.cs`、`LiveSubtitleTranscriber.cs`、`LiveSubtitleTranslator.cs`、`LiveSubtitleChunker.cs`、`LiveSubtitleAudioCapture.cs`、`LiveSubtitleSerialClock.cs`、`LiveSubtitleModelPath.cs`（中文路径模型自动拷到纯英文路径）。
- 配置（`%APPDATA%\TiviMateWin\config.json`）：LiveSubtitleEnabled、LiveSubtitleSerial（串行，默认 true）、LiveSubtitleModel（1.5B）、LiveSubtitleWhisperModel（tiny.en）、LiveSubtitleWindowSeconds（1.5 秒）。
- v1.4.30 调优：Whisper 关温度回退（`WithTemperature(0f)`、`WithTemperatureInc(0f)`）、单段（`WithSingleSegment()`）、每段 64 token（`WithMaxTokensPerSegment(64)`）、单窗 6 秒超时（超时放弃当窗、不卡喂音频循环）。效果：掉窗率约 21%（v1.4.29 是 36–38%）、transcribeTimeouts=0、样本内无 5–12 秒停滞、端到端典型 1.7–2.8 秒。
- v1.4.31 修复（关字幕崩溃）：Whisper 处理器释放必须用 `await using` 异步释放，听写器/翻译器 Dispose 前要等在途转写/翻译任务收尾（等门，带超时）。同步释放会在转写中途崩进程：ggml-cpu-whisper.dll 原生访问违规 0xc0000005，无托管异常、无日志。台式机复测被对话框阻塞挡住未验证，代码+单元测试（92/92，新加 3 个释放安全测试）+CI 已验证。
- 日志看门狗每 30 秒一行：`live subtitle watchdog: stage=... received=... dropped=... transcribed=... translated=... transcribeTimeouts=... lastTranscribe=...ms lastTranslate=...ms lastEndToEnd=...ms`。
- 依赖最新情况（2026-11-18 查过）：Whisper.net 1.9.1、LLamaSharp 0.30.0、LibVLCSharp.WPF 3.10.2、NAudio 3.3.0、VideoLAN.LibVLC.Windows 3.0.24 均已是最新稳定版；Svg.Skia 1.0.0 → 5.2.3 已升（`SKSvg` 类型仍在，只影响台标光栅化）。

## 播放与重连（已知行为）

- 播放列表：`https://live.ciweihaha.win/live.m3u`；源是腾讯云函数 SCF，会抖动（playlist 返回 200 但分片可能断）。
- EPG：`ExtractEpgUrl` 除了 `url-tvg=` 还要认 `x-tvg-url=`（两者都有时 `url-tvg=` 优先）。直播源首行是 `x-tvg-url="https://ysp.ciweihaha.win/epg.xml"`，只认前者时 EPG 永远加载不到。v1.4.31 已修并验证：日志 `epg loaded 56 channels`。
- 断流自动重连：v1.4.31 已修好上限泄漏——触顶后 5 分钟降频（不重试、不刷 ERROR），触顶 ERROR 同频道 10 分钟只记一次（已验证：CCTV综合触顶 ERROR 只记一次，降频期内无重试、无 ERROR）。残留：看门狗 INFO `unexpected stop detected, reconnect` 在降频期内仍每 8 秒刷一条——日志在调重连前就记了，实际没重试，措辞误导，属日志噪音，未修。
- 无同名备用源时失败 4–5 次会跳下一台（SkipNext）；同名备用源优先（`FindBackupChannel`）。
- 硬件解码：配置 `HardwareDecode` 为 false（软解）；连续失败时有一次硬解回退机会（会弹框重启）。

## 两台机器

- 云电脑优先，台式电脑兜底。云电脑：天翼云，Tailscale 100.123.171.106，用户 Administrator；正式版 `C:\TiviMateWin`，测试版 `C:\TiviMateWin-Test\app`；计划任务 RunTiviMateWin 不要删。台式机：DESKTOP-MI11M34，Tailscale 100.109.57.109，用户 jiaheng（profile 路径含中文），Win11 Canary 26300；正式版 `C:\TiviMateWin`；计划任务 TiviMateWin-Start 不要删。
- 云电脑↔台式机的 Tailscale 走中继，链路极慢且会恶化：160MB 安装包直连 GitHub 超时、经台式机代理约 8KB/s、台式机直传约 9.5KB/s 后降到 0.75KB/s。传大文件要用断点续传循环在后台慢慢跑（云电脑计划任务 `TiviMateWin-DL1430`，`C:\tmw-resume.ps1`，代理 + `curl -C -`，6 小时循环，下完写 `C:\tmw-resume.log`）。
- WinRM：`WINRM_PASS=123 ~/workspace/.venv-winrm/bin/python ~/workspace/dt.py [desktop|cloud] "<powershell>"`，单条命令约 3000 字符上限；通道常断（ProxyError / ReadTimeout），断了等 60 秒重试，不要反复刷。GUI 应用不要用 WinRM 直接起，用计划任务在控制台会话起。
- WinRM 大坑：经 bash 双引号传命令时 `$变量` 会被 bash 展开吃掉（含 `$` 的复杂命令一律写文件后 `dt.py -f` 发送）。
- WinRM 里 `Invoke-WebRequest` 报 WebCmdletIEDomNotSupportedException，下载一律用 `curl.exe -x http://127.0.0.1:10809`（台式机代理）。

## 测试经验（v1.4.31 实测）

- 键盘焦点在搜索框（TextBox）时，应用全局键（方向/数字/回车）全部被忽略——测换台前先鼠标点一下列表项，把焦点移回列表。
- 收藏开启时按钮改名“★ 收藏中”，测试脚本要按状态找按钮，别按旧名“收藏”找。
- 模态对话框打不开不一定是新版回归——要拿旧版解到单独目录对照测：v1.4.30 与 v1.4.31 的对话框代码逐字节相同，v1.4.30 对照版同样打不开（而 1.4.30 全功能测试当天 19:35–20:35 对话框都能开），说明是台式机/会话环境问题，不是代码回归。下一步：重启台式机再试。
- 应用可能悄悄退出且无任何崩溃记录（无 Application Error、无 .NET Runtime 事件、无 WER 报告、应用日志直接断在字幕/播放行）——要单独盯进程存活（定时查 PID），别只看崩溃日志。字幕开关与退出无关；10 分钟监视曾活满 600 秒，不是必然死。

## 已知遗留问题

- 模态对话框阻塞（2026-11-18 晚，环境问题，非 1.4.31 回归）：设置/添加源/网盘/分组管理/MessageBox 全部打不开，UIA Invoke、鼠标点击、键盘回车三种方式都无效，无异常、无日志、无事件。设置窗口（4 复选框/3 下拉/1 滑杆/保存）、主题切换、频道 logo、右键菜单（7 项）、多窗口、播放/暂停、停止、音量、托盘图标因此未测。下一步：重启台式机（等平板测试结束再重启——重启会断台式机 adb）再试。
- 应用间歇性悄悄退出（2026-11-18 晚）：启动后 1–15 分钟内不定期消失，无任何崩溃记录。已排除：用户没在用台式机、Defender 0 条相关事件、计划任务、其他 agent 干扰、硬件解码回退。
- 左栏空白（未定性，v1.4.28 和 v1.4.30 都有）：截图里左侧频道列表栏纯白/纯黑、一个字都没有；但 UIA 探查确认列表有 14 个频道、可点按，应用其余功能正常。主题文件完好、配置是深色、无主题报错。可能是在用户不坐电脑前、经计划任务启动时的渲染/合成问题。需在用户坐电脑前对照确认。
- 小窗：UIA 点“小窗”后弹出 400×268 浮动窗、主窗口最小化；关闭小窗后主窗口恢复（v1.4.30 已验证）。托盘图标未直接验证（v1.4.31 复测中也未直接验证，仅代码+日志部分确认）。
- 截图方法：自建管线 `~/workspace/automation/shot-desktop-cmd.ps1`（全屏）、`shot-desktop-cmd2.ps1`（先置前再截）、`shot-window-cmd.ps1`（PrintWindow 截窗口本身），均经 `dt.py -f` 发送；UIA 探查脚本在 `~/workspace/automation/uia-desktop-cmd.ps1`。
