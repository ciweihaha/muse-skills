# 全自动迭代发布循环

## 循环规则

- 用户 2026-11-18 下令：全部项目切成全自动化工作流，云端编译发新版本，电脑平板测试，深入审查修复 bug，修复发新版本，一直循环，没有 bug 就升级算法、提升效率稳定性、降低内存占用、评估添加新功能，需要的话自己添加发版，也按这个流程来。

- 永远循环，不停。旧的连续两轮无可修内容就停的规则作废。

- 修好就发版，不用等用户下令。每轮走审查、修或者优化、本地构建加测试全过、升版本、推代码、打 tag、核对 Release 与安装包、在对应机器上装好实测、记录。

- 平时不汇报。只有卡住、需要用户出面才能解决的事，才写进 blockers.md，并在那一轮的最终消息里写清楚。

- 不要删仓库。任何情况下都不删仓库。

- 不要和正在跑的活撞车。每轮开始前先查正在跑的子代理，跳过已有 worker 在干活的项目，同一项目不要重复开一轮。一轮做不完就把进度写进 state.json 标 in_progress，下一轮接着做。

## 状态文件

- 循环状态在 `~/workspace/automation/state.json`，里面有 queue 顺序、next_index、每个项目的 last_round、last_release、status、version_code、note，以及 installed 清单和 blockers。

- 规则全文在 `~/workspace/automation/LOOP.md`，先读它再动手。

- 每轮写一个日志到 `~/workspace/automation/runs/<日期>-<项目>-<内容>.md`，写版本号、tag、Release 链接、安装包大小与 SHA、测试结果。

- 阻塞事项写进 `~/workspace/automation/blockers.md`，只记需要用户出面的事。

- 定时任务是 `auto-loop-all-projects`，每 6 小时跑一次。

## 每轮算法

- 读 state.json，查正在跑的子代理，记下正在跑的 worker 都在哪个项目。

- 按 state.json 的 queue 顺序挑下一个项目。跳过 TVBoxOS，它的远程仓库已删。跳过有 worker 在跑的项目。跳过 state 里标了 blocker 的项目。

- 给这个项目派一个 worker 子代理，任务书必须自带全部上下文。

- 等 worker 回报后，更新 state.json 里这个项目的 last_round、last_release、status 和 note，再写 runs 日志。

- 推进 queue 指针到下一个项目，回到第一步，等下一轮触发时继续。

## Worker 任务书模板

- 任务书照抄下面的结构，只填项目信息。

- 写清仓库、分支、本地克隆路径、当前最新 Release、本地 HEAD。

- 按顺序列出本轮要做的事。先深入审查：依赖升最新、找 bug、看性能、内存和稳定性短板、评估有没有必要的新功能，列出本轮要修和要做的清单。

- 修 bug，有就修，没有就做优化或者判断有必要的新功能。改完本地构建加全部测试必须全过，按本项目规则来。

- 升版本，按本项目版本文件来。推代码，走 PAT，见 github-pat-ops skill。打 tag，触发 Release 构建。核对 Release 已发布、安装包大小与 SHA 对得上。

- 在对应机器上装好实测，写日志。平板一次只能一个 agent 用，抢前台会打断另一边的测试，先确认平板空闲。

## Available connectors

- [GitHub](/opt/hatch/connectors/github/CONNECTOR.md) for reading repos, issues and pull requests during review.

- [Github Pat](~/workspace/custom-connectors/custom.github-pat/CONNECTOR.md) for pushing code, creating tags and releases through the PAT helpers.

- Use `muse.connector_search` to find other connectors and `muse.skill_search` to find related skills.

- If nothing here fits, look on the web for a maintained API or MCP server and set it up with [Custom Connectors and MCPs](/opt/muse/skills/custom-connector-setup/SKILL.md), or build a custom connector.

## Alternatives for better results

- 一轮里同时发现 bug 和优化点时，先修 bug 再做优化，优化不要和 bug 修复混在一次提交里。

- 构建失败先看 CI 日志再动手，不要反复重试同一个失败命令。
