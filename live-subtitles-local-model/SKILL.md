# 本地小模型实时字幕

## 模型选择

- 翻译模型用 qwen2.5:1.5b，不到 1GB，986 MB，是用户选的折中。最小是 qwen2.5:0.5b 约 400MB，原默认 qwen2.5:3b 约 1.9GB。

- 结论是 1.5B 已够字幕用，不用更大模型。字幕真需要更大模型时，先小成本验证，租云 GPU 试 32B 翻译效果，或者台式机加一张二手 3090，再决定。

- 听写用 whisper tiny.en。翻译热机约 1.1 秒一行，冷启动第一行约 10 秒。

## Ollama 字幕翻译

- 云电脑已装好 Ollama 并拉好 qwen2.5:1.5b，地址是 `http://127.0.0.1:11434`。OllamaServe 定时任务在登录时启动，保证 Ollama 一直跑着。

- UnBox 里模型名已改成 qwen2.5:1.5b，点检测模型通过会显示本地模型就绪，可以开字幕翻译。

- Ollama 只留给 UnBox 字幕翻译用，TiviMateWin 不用 Ollama。

## UnBox 直播实时字幕

- 链路是前端 Web Audio 抓 video 声音，转 16kHz 单声道，后端按 1.5 秒窗口送 whisper-server 听写英文，再经 Ollama qwen2.5:1.5b 翻中文，最后用 Wails 事件叠到画面上。

- 实测端到端约 3.3 秒，听写 0.7 秒加翻译 1.1 秒加 1.5 秒窗口。配置保持 tiny.en 和 1.5 秒不变。

- Whisper 程序加模型首次使用时自动下载，约 85 MB，缓存下来，不打进安装包。

- 只在 Web 播放器后端生效，mpv 后端没有 video 可抓。云电脑 mpv 已坏，缺 DLL，跑不起来。

- 测试版在云电脑 `C:\UnBox-Test\UnBox-live-subtitles-windows-amd64.exe`，开关已开。画面上的字幕目视确认要在交互式桌面打开播 BBC 才能亲眼确认，WinRM 非交互会话里 WebView2 起不来。

## TiviMateWin 直播实时字幕

- 模型进程内跑，用 LLamaSharp 加载 Qwen2.5-1.5B GGUF，加 Whisper.NET 听写，不用 Ollama，不用后台服务。模型文件首次打开实时字幕时下载缓存，共约 1.26GB，GGUF 约 1.04GB，不打进安装包。

- 直播没有字幕轨道，要 Whisper 听写英文加模型翻译加画面叠字幕。目标是字幕比说话慢不超过 6 秒。

- v1.4.29 已修好两个问题。一是中文路径模型加载，新增 Services/LiveSubtitleModelPath.cs，自动把模型拷到纯英文路径，任何中文用户名机器都自动修好。二是延迟，从约 11 秒降到中位数云电脑 2.4 秒、台式机 2.9 秒，用 tiny.en、1.5 秒窗口、MaxTokens 32、串行默认。中文输出云电脑 47 条全中，台式机 73 条全中，翻译质量好。

- 配置键是 LiveSubtitleEnabled、LiveSubtitleSerial、LiveSubtitleModel、LiveSubtitleWhisperModel、LiveSubtitleWindowSeconds。

- 已知残留。约 4% 到 13% 的窗口 ASR 卡 5 到 12 秒，会自己恢复，原因未定位。串行模式下约 36% 到 38% 的窗口被丢弃，语音覆盖约 60%。

- 开发在单独分支，不动 main，不影响发版验证。先在云电脑上测，云电脑解决不了再上台式机。

## 测试方法

- 端到端延迟要在云电脑和台式机两台机器上都测，取中位数和 p90，不要只测一次就下结论。

- 中文输出要数条数，确认每条都是中文且翻译准确。

- 性能不达标时先看 CPU 推理吞吐，翻译质量本身好不等于延迟够。

## Available connectors

- Use `muse.connector_search` to find other connectors and `muse.skill_search` to find related skills.

- If nothing here fits, look on the web for a maintained API or MCP server and set it up with [Custom Connectors and MCPs](/opt/muse/skills/custom-connector-setup/SKILL.md), or build a custom connector.

## Alternatives for better results

- 字幕延迟超标时先缩窗口和 MaxTokens，再考虑换模型，不要一上来就换大模型。

- 首次播放的字幕要单独测一次，首播链路的 bug 和稳定播放时的 bug 不一样。
