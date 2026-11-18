# 电视和电脑应用构建、测试与发版

## 通用流程

- 每轮先在本地构建加全部测试，全部通过再升版本、推代码、打 tag、发版。

- 发版前核对 Release 已发布，安装包大小与 SHA-256 对得上，再装到机器上实测。

- 安装包先放到台式机 `D:\apks`，核对大小无误后再装。平板用 adb 安装，电脑从 Release 压缩包安装。

- 实测要截图留证，截图用 `~/workspace/automation/shot.py`。平板一次只能一个 agent 用，抢前台会打断另一边的测试，先确认平板空闲。

- 写 runs 日志，写版本号、tag、Release 链接、安装包大小与 SHA、测试结果，再更新 state.json。

## OrionTV RN 影视 app

- 仓库 `ciweihaha/OrionTV-build`，本地克隆 `~/workspace/oriontv-build`，分支 master，包名 `com.oriontv`。

- 测试用 jest、tsc、eslint、expo export，全部通过才算过。最新发版是 v1.7.6，versionCode 10706，305 个测试。

- 点播走写死的聚合地址 `https://ysp.ciweihaha.win/agg`，直播默认 `https://live.ciweihaha.win/live.m3u`。应用里没有 vod.json 导入，只有一个 MoonTV 地址栏，配 MoonTV 会弹密码登录框，密码未知，不要去配。

- 平板验证要点：点分类前先用 uiautomator 确认按钮中心坐标，点偏到标题栏会造成零流量误判。点后看选中状态、流量变化和报错内容。报错要看清具体错误种类，不要只报加载失败。

## OrionTV-native Kotlin 原生版

- 仓库 `ciweihaha/OrionTV-native`，本地克隆 `~/workspace/oriontv-native`，分支 main，包名 `com.oriontv`，debug 包是 `com.oriontv.debug`，可以和正式版共存。

- 本地构建环境在 `~/workspace/.jdk17` 和 `~/workspace/.android-sdk`。每次构建前重建信任库，因为出口 CA 约一小时轮换一次。Gradle 要设 `GRADLE_USER_HOME`、`JAVA_HOME`、`ANDROID_HOME`，加 `-Djava.net.preferIPv4Stack=true`，`LANG=C.UTF-8`，`JAVA_TOOL_OPTIONS` 里塞信任库路径和 `-Dfile.encoding=UTF-8`。跑完一次构建就把自己 GRADLE_USER_HOME 下的 daemon 杀掉，否则下一次构建会报锁超时。

- `~/workspace/.android-sdk/platforms/android-37` 是 android-37.2 复制改名的，仅供本地编译验证，不要删。

- 测试 667 个通过才算过。最新发版是 v2.0.8，versionCode 10711。正式版 APK 约 3 MB。

## FongMi

- 仓库 `ciweihaha/TV`，分支 fongmi，本地克隆 `~/workspace/fongmi-tv`。

- 58 个单元测试全过，压力测试全过。Lottie 已在第 133 轮整个去掉，不要再引进来。

- 常规轮次只推代码不升版，除非本轮要发版。最新发版是 v5.9.8。

## TVBoxOS

- 本地克隆只有 `~/workspace/tvboxos`，远程仓库用户已删，GitHub 返回 404，无法推送。

- 只做本地提交，不要重建远程、不要推、不要发版。

## mytv 天光云影

- 仓库 `ciweihaha/mytv-android-tgyy`，本地克隆 `~/workspace/mytv-android-tgyy`，分支 main，包名 `top.yogiczy.mytv`。

- 构建用 `./gradle-mytv.sh`，674 个测试全过才算过。最新发版是 v2.2.25，versionCode 18，APK 约 5 MB。

- 每次发版前同步上游提交，能干净合入就合入。直播断线处理走应用侧自动重连，见 live-source-phoenix skill。

## UnBox

- 仓库 `ciweihaha/UnBox`，本地克隆 `~/workspace/unbox`，Go 加 Wails v3.1.0。

- 规则：TDD，公开错误和注释写中文，TypeScript 保持 6.x，绝不升 7.x，绝不跑 `go mod tidy`。发版就是推一个 `v*` tag。

- 云电脑安装位置是 `C:\Program Files\RejectCode\UnBox`。发布构建的 macOS 任务可能因 GitHub 容量问题反复取消，取消后重跑失败任务再验证。

## TiviMateWin

- 仓库 `ciweihaha/TiviMateWin`，本地克隆 `~/workspace/tivimatewin`，.NET 11 WPF。

- 版本唯一来源是 csproj 里的 `<Version>`。发版流程是改版本、推代码、打 tag。

- 先在云电脑上测，云电脑解决不了再上台式机。云电脑只测 Windows 应用，不装模拟器。

- 从 Release 压缩包安装到 `C:\TiviMateWin`。托盘图标和小窗关闭是已验证过的回归点，每次发版后重测一遍。

## Available connectors

- [GitHub](/opt/hatch/connectors/github/CONNECTOR.md) for reading repos and checking workflow runs.

- [Github Pat](~/workspace/custom-connectors/custom.github-pat/CONNECTOR.md) for pushing code, creating tags and releases through the PAT helpers.

- Use `muse.connector_search` to find other connectors and `muse.skill_search` to find related skills.

- If nothing here fits, look on the web for a maintained API or MCP server and set it up with [Custom Connectors and MCPs](/opt/muse/skills/custom-connector-setup/SKILL.md), or build a custom connector.

## Alternatives for better results

- 平板测不出的滚动问题，最终要在用户电视上验证，不要在平板上反复磨。

- 发版后立刻在真机上装一遍，不要只看构建绿就以为发好了。
