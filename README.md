# muse-skills

Muse 的技能库：把做过的项目、设备和服务写成可复用的操作手册，方便其他智能体随时读取和复用。

规则：每完成一个项目，就把它做成 skill 并上传到这里；本地 `~/workspace/custom-skills/` 和这个仓库保持同步。Skill 里不写密码、token、密钥，只写走环境变量或安全存储。

## Skills

| Skill | 说明 |
|---|---|
| [device-connections](device-connections/SKILL.md) | 连接台式机、云电脑和安卓平板（Tailscale、WinRM、adb、代理） |
| [auto-release-loop](auto-release-loop/SKILL.md) | 全自动迭代发布循环：云端编译、发版、实测、修 bug，一直循环 |
| [tv-app-build-release](tv-app-build-release/SKILL.md) | 电视和电脑应用构建、测试与发版（OrionTV、FongMi、mytv、UnBox、TiviMateWin） |
| [live-source-phoenix](live-source-phoenix/SKILL.md) | 直播源与凤凰秀：M3U、腾讯云函数、官方源 token、签名过期 |
| [live-subtitles-local-model](live-subtitles-local-model/SKILL.md) | 本地小模型实时字幕（Ollama、Whisper、UnBox、TiviMateWin） |
| [tailscale-network](tailscale-network/SKILL.md) | Tailscale 与网络：直连、中继、UPnP、DERP、路由器 |
| [lan-iot-scan](lan-iot-scan/SKILL.md) | 局域网与物联网扫描：认设备、健康检查 |
| [desktop-cleanup](desktop-cleanup/SKILL.md) | 台式机清理维护：卸载 WSA、Android Studio、释放空间 |
| [github-pat-ops](github-pat-ops/SKILL.md) | GitHub PAT 操作：绕过 GitHub App 权限、打 tag、发版 |

每个 skill 目录下有 `SKILL.md`（操作手册）和 `skill.yaml`（元数据）。
