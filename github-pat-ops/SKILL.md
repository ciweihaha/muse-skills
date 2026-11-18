# GitHub PAT 操作

## 为什么要用 PAT

- GitHub App "Meta Muse AI" 没有 Workflows 权限，推 `.github/workflows/*` 会返回 403。只有应用所有者 Meta 能授权，重试过，依然 403。

- 绕法是用经典 PAT，带 repo 和 workflow 权限，走 custom.github-pat 连接器。PAT 本身存在安全存储里，助手用注册连接取用。

- 绝不把 PAT、token、密钥、密码写进文件、命令参数、环境变量或日志里。助手只调已注册的连接。

## 助手脚本

- 助手脚本都不是可执行文件，永远用 `python3 <路径>` 调用。

- 读接口用 `~/workspace/fongmi-tv-aar-inspect/github_api_get.py "<api路径>"`，api 路径必须以 `/` 开头。

- 批量推文件用 `~/workspace/fongmi-tv-aar-inspect/github_push_batch.py <owner> <repo> <branch> <message> <文件...>`。从仓库根目录跑，文件用相对路径。提交信息里不要带 commit hash。

- 打 tag 用 `~/workspace/fongmi-tv-aar-inspect/github_create_tag.py`，要传完整 40 位 SHA。打 tag 会触发 `push: tags` 的工作流。

- 其他助手都在同一目录。`github_create_release.py` 建 Release，`github_dispatch.py` 触发 workflow_dispatch，`github_download_artifact.py` 下载构建产物，`github_job_log.py` 查任务日志，`github_rerun_failed.py` 重跑失败任务，`github_reset_branch.py` 重置分支，`github_set_mode.py` 改模式，`github_push_batch_mixed.py` 混合批量推。

- Fork 用 `~/workspace/custom-connectors/custom.github-pat/bin/github_fork_repo.py <组织>/<仓库> [新名字]`，调 `POST /repos/{owner}/{repo}/forks`，成功返回 202，重复返回 422。Fork 前先查 `/repos/ciweihaha/<名字>` 是否已存在。

- 单文件写入用 `~/workspace/custom-connectors/custom.github-pat/bin/github_put_file.py`，走 GitHub Contents API，workflow 文件也能推。

## 发版流程

- 改版本、推代码、打 tag、触发 Release 构建。打 tag 要用完整 40 位 SHA，不要用短 SHA。

- 核对 Release 已发布，安装包大小与 SHA-256 对得上，再装到机器上实测。

- Release 附件用 API 上传会返回 401，改用 Release 工作流或者浏览器上传。

- 发布构建的 macOS 任务可能因 GitHub 容量问题反复取消，这是已知问题。取消后重跑失败任务再验证，四个平台成功、macOS 成功后 Release 步骤会自动创建 Release。

## 失败后先读再重试

- 请求失败或超时后，先用读接口查当前状态再重试，因为第一次请求可能已经完成了。

- 401 或 403 先检查请求本身，确认助手挂了凭证、主机已注册、权限范围对得上。只有带凭证的请求仍被拒绝时，才重连。

## Available connectors

- [GitHub](/opt/hatch/connectors/github/CONNECTOR.md) for reading repos, issues, pull requests and workflow runs.

- [Github Pat](~/workspace/custom-connectors/custom.github-pat/CONNECTOR.md) for pushing code, creating tags and releases, and forking repos through the PAT helpers.

- Use `muse.connector_search` to find other connectors and `muse.skill_search` to find related skills.

- If nothing here fits, look on the web for a maintained API or MCP server and set it up with [Custom Connectors and MCPs](/opt/muse/skills/custom-connector-setup/SKILL.md), or build a custom connector.

## Alternatives for better results

- 推 workflow 文件一律走 PAT，不要再试 GitHub App，已经确认过 403。

- 打 tag 前先确认 SHA 是要发的那次提交，不要打错。
