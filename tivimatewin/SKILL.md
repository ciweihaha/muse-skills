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
- 最新版：v1.4.30（2026-11-18，提交 c5921db9，Release https://github.com/ciweihaha/TiviMateWin/releases/tag/v1.4.30 ，`TiviMateWin-win-x64.zip` 160,869,322 字节，SHA-256 `ef066c8a7a67de4f2610a31ded2908db5f9e31fb7b58f32354fca4f301fd4a69`）。

## 构建与测试

- .NET 11 SDK：台式机 `C:\dotnet11`（11.0.100）；本机 Linux `~/workspace/.dotnet11`（11.0.100）。Linux 上 restore Windows 目标需 `-p:EnableWindowsTargeting=true`。
- 本机只有 1.5 共享 CPU，测试跑不动（vstest 超时）；以台式机构建+测试与 CI 为准。
- 台式机构建在 `D:\build\TiviMateWin-main`（从 main zipball 解压）：`dotnet build -c Release`、`dotnet test`（v1.4.30 是 89/89 全过）。
- `--selftest` 经 WinRM 挂起（非交互会话环境问题，不计为失败；CI 的自测是权威验证）。
- 2 个已有的 NU1510 警告：ProtectedData 包在 .NET 11 里自动可用，建议移除引用。

## 实时字幕管线（进程内，无 Ollama）

- 链路：音频抓取 → Whisper.NET（tiny.en）听写 → LLamaSharp（Qwen2.5-1.5B GGUF）翻译 → 画面叠字幕。模型首次打开时下载缓存（共约 1.26GB），不打进安装包。
- 关键文件：`Services/LiveSubtitleService.cs`、`LiveSubtitlePipeline.cs`、`LiveSubtitleTranscriber.cs`、`LiveSubtitleTranslator.cs`、`LiveSubtitleChunker.cs`、`LiveSubtitleAudioCapture.cs`、`LiveSubtitleSerialClock.cs`、`LiveSubtitleModelPath.cs`（中文路径模型自动拷到纯英文路径）。
- 配置（`%APPDATA%\TiviMateWin\config.json`）：LiveSubtitleEnabled、LiveSubtitleSerial（串行，默认 true）、LiveSubtitleModel（1.5B）、LiveSubtitleWhisperModel（tiny.en）、LiveSubtitleWindowSeconds（1.5 秒）。
- v1.4.30 调优：Whisper 关温度回退（`WithTemperature(0f)`、`WithTemperatureInc(0f)`）、单段（`WithSingleSegment()`）、每段 64 token（`WithMaxTokensPerSegment(64)`）、单窗 6 秒超时（超时放弃当窗、不卡喂音频循环）。效果：掉窗率约 21%（v1.4.29 是 36–38%）、transcribeTimeouts=0、样本内无 5–12 秒停滞、端到端典型 1.7–2.8 秒。
- 日志看门狗每 30 秒一行：`live subtitle watchdog: stage=... received=... dropped=... transcribed=... translated=... transcribeTimeouts=... lastTranscribe=...ms lastTranslate=...ms lastEndToEnd=...ms`。
- 依赖最新情况（2026-11-18 查过）：Whisper.net 1.9.1、LLamaSharp 0.30.0、LibVLCSharp.WPF 3.10.2、NAudio 3.3.0、VideoLAN.LibVLC.Windows 3.0.24 均已是最新稳定版；Svg.Skia 1.0.0 → 5.2.3 已升（`SKSvg` 类型仍在，只影响台标光栅化）。

## 播放与重连（已知行为）

- 播放列表：`https://live.ciweihaha.win/live.m3u`；源是腾讯云函数 SCF，会抖动（playlist 返回 200 但分片可能断）。
- 断流自动重连：`TryAutoReconnect` 的 5 次上限只管 EndReached 这一路；停滞看门狗（`OnStallWatchTick`）和 `HandlePlayError` 直接调 `PlayAsync`，不经过计数——所以死流会每 4–5 秒无限重试，`auto-reconnect stopped after limit` 每 8 秒记一次 ERROR。好处是源一恢复就自动回来；下轮可修：统一计数、超限后降频重试、重复 ERROR 降级。
- 无同名备用源时失败 4–5 次会跳下一台（SkipNext）；同名备用源优先（`FindBackupChannel`）。
- 硬件解码：配置 `HardwareDecode` 为 false（软解）；连续失败时有一次硬解回退机会（会弹框重启）。

## 两台机器

- 云电脑优先，台式电脑兜底。云电脑：天翼云，Tailscale 100.123.171.106，用户 Administrator；正式版 `C:\TiviMateWin`，测试版 `C:\TiviMateWin-Test\app`；计划任务 RunTiviMateWin 不要删。台式机：DESKTOP-MI11M34，Tailscale 100.109.57.109，用户 jiaheng（profile 路径含中文），Win11 Canary 26300；正式版 `C:\TiviMateWin`；计划任务 TiviMateWin-Start 不要删。
- 云电脑↔台式机的 Tailscale 走中继，链路极慢且会恶化：160MB 安装包直连 GitHub 超时、经台式机代理约 8KB/s、台式机直传约 9.5KB/s 后降到 0.75KB/s。传大文件要用断点续传循环在后台慢慢跑（云电脑计划任务 `TiviMateWin-DL1430`，`C:\tmw-resume.ps1`，代理 + `curl -C -`，6 小时循环，下完写 `C:\tmw-resume.log`）。
- WinRM：`WINRM_PASS=123 ~/workspace/.venv-winrm/bin/python ~/workspace/dt.py [desktop|cloud] "<powershell>"`，单条命令约 3000 字符上限；通道常断（ProxyError / ReadTimeout），断了等 60 秒重试，不要反复刷。GUI 应用不要用 WinRM 直接起，用计划任务在控制台会话起。
- WinRM 大坑：经 bash 双引号传命令时 `$变量` 会被 bash 展开吃掉（含 `$` 的复杂命令一律写文件后 `dt.py -f` 发送）。
- WinRM 里 `Invoke-WebRequest` 报 WebCmdletIEDomNotSupportedException，下载一律用 `curl.exe -x http://127.0.0.1:10809`（台式机代理）。

## 已知遗留问题

- 左栏空白（未定性，v1.4.28 和 v1.4.30 都有）：截图里左侧频道列表栏纯白/纯黑、一个字都没有；但 UIA 探查确认列表有 14 个频道、可点按，应用其余功能正常。主题文件完好、配置是深色、无主题报错。可能是在用户不坐电脑前、经计划任务启动时的渲染/合成问题。需在用户坐电脑前对照确认。
- 小窗：UIA 点“小窗”后弹出 400×268 浮动窗、主窗口最小化；关闭小窗后主窗口恢复（v1.4.30 已验证）。托盘图标未直接验证。
- 截图方法：自建管线 `~/workspace/automation/shot-desktop-cmd.ps1`（全屏）、`shot-desktop-cmd2.ps1`（先置前再截）、`shot-window-cmd.ps1`（PrintWindow 截窗口本身），均经 `dt.py -f` 发送；UIA 探查脚本在 `~/workspace/automation/uia-desktop-cmd.ps1`。
