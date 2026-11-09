# 10 人团队 SAS CI/CD 练习（GitLab + 你电脑上的 SAS）

`simulate_team.py` 扮演 10 个开发人员：每人建分支、以自己的名字提交 SAS 代码、开合并请求（MR）。
你扮演维护者：看流水线、审代码、要求修改、解决冲突、合并。
流水线先用 Python 检查代码规范（GitLab 云端），再在**你自己的 Windows 电脑上真的运行 SAS**。

## 一、准备 GitLab 项目

1. **令牌**：GitLab → 头像 → Edit profile → Access tokens → 新建令牌，勾选 **api** 权限。
2. **项目设置**：
   - Settings → Merge requests → 勾选 **Pipelines must succeed**
   - Settings → Repository → Protected branches → `main`：Allowed to push = **No one**，Allowed to merge = Maintainers

## 二、把你的电脑注册成 GitLab Runner

**前提**：电脑上已装 Git for Windows（PowerShell 里运行 `git --version` 能显示版本号）。

1. **在 GitLab 创建 Runner**：项目 → Settings → CI/CD → Runners → **New project runner**
   - Tags 填：`sas,windows`
   - **不要**勾选 Run untagged jobs（否则云端的 Python 检查任务会跑到你电脑上）
   - 点 Create runner，复制页面上的 `glrt-...` 令牌
2. **安装并注册**：用**管理员身份**打开 PowerShell：
   ```powershell
   mkdir C:\GitLab-Runner
   cd C:\GitLab-Runner
   Invoke-WebRequest -Uri "https://s3.dualstack.us-east-1.amazonaws.com/gitlab-runner-downloads/latest/binaries/gitlab-runner-windows-amd64.exe" -OutFile gitlab-runner.exe
   .\gitlab-runner.exe register --non-interactive --url "https://gitlab.com" --token "glrt-你的令牌" --executor "shell" --shell "powershell" --description "my-sas-pc"
   ```
   `--shell "powershell"` 很重要：新版默认用 pwsh（PowerShell 7），很多电脑没装，会报 `"pwsh": executable file not found`。
3. **启动 Runner**（练习时推荐这种）：
   ```powershell
   .\gitlab-runner.exe run
   ```
   保持这个窗口开着，Runner 就在线；关掉窗口就离线。任务以你的 Windows 账号运行，SAS 最不容易出权限问题。
   想开机自动运行，可以装成服务：`.\gitlab-runner.exe install --user ".\你的用户名" --password "你的密码"`，再 `.\gitlab-runner.exe start`。
4. **确认在线**：Settings → CI/CD → Runners，看到你的 Runner 旁边是绿点。
5. **打开 SAS 任务**：Settings → CI/CD → Variables → Add variable
   - `SAS_RUNNER` = `yes`，**取消勾选 Protect variable**（不然 MR 分支读不到它，SAS 任务会被跳过）
   - 如果 sas.exe 不在 `C:\Program Files\SASHome\SASFoundation\9.4\sas.exe`，再加 `SAS_EXE` = 实际路径

**可选的本地预检**：把仓库 clone 到电脑上，在仓库目录运行
`powershell -ExecutionPolicy Bypass -File ci\run_sas.ps1`，看 `ci_out\summary.txt` 和 `ci_out\sas.log`。

## 三、运行团队模拟

在你电脑上（Python 3.9+）：
```
pip install python-gitlab
set GITLAB_TOKEN=glpat-你的令牌
set GITLAB_PROJECT=lsu-group2092945/zj_ci_cd
python simulate_team.py --team
```
任何命令后面加 `--dry-run` 只预览，不改 GitLab。

| 阶段 | 命令 | 发生什么 | 你练习什么 |
|---|---|---|---|
| 0 | `--wave 0` | 一个 MR 加入流水线文件和 01/02 程序 | 第一次绿色流水线，合并它 |
| 1 | `--wave 1` | Ava、Ben、Hana 提交 3 个正确的功能 | 审代码、批准、合并多个 MR |
| 2 | `--wave 2` | Chen（少 `run;`）、Dana（写死路径）、Ivan（没文件头）**lint 失败**；Jia（列名拼错）**lint 通过但 SAS 报错** | 看失败日志；理解为什么既要 lint 又要真跑 SAS |
| 修复 | `--fix chen` / `dana` / `ivan` / `jia` | 开发人员推送修复 | 流水线自动重跑变绿 |
| 3 | `--wave 3`（先合并阶段 1） | Eli 和 Fay 改同一行 README；Gus 把 `class_clean` 改名为 `class_prep` | 在 GitLab 里解决冲突；逻辑冲突被流水线拦住 |
| 修复 | `--fix gus` | Gus 变基并更新所有引用旧名字的程序 | 跨文件的协同修改 |

## 四、流水线在做什么

| 任务 | 在哪跑 | 做什么 |
|---|---|---|
| `sas_lint` | GitLab 云端 | 检查团队规范：文件头、无写死路径、每个 DATA/PROC 都有 `run;`/`quit;`、读取的 WORK 数据集必须由前面的程序创建 |
| `sas_run` | 你的电脑 | 在**同一个** SAS 会话里按文件名顺序 `%include` 所有 `NN_*.sas`，日志里有 `ERROR` 就失败 |
| `deploy_local` | 你的电脑，只在 main 上 | 把程序复制到 `C:\sas_prod\releases\<提交号>`，并在 `CURRENT.txt` 记下当前版本 |

在 GitLab 里看结果：
- **MR → Pipelines → sas_run**：日志末尾有每个程序的 PASS/FAIL
- **MR 的 Tests 标签**：每个 SAS 程序一个测试结果
- **任务页右侧 Browse**：下载 `ci_out/`，里面有 `sas.log`、`results.html`（输出表格和图）
- **Operate → Environments → production**：每次部署的记录；点旧版本的 Re-deploy 就是**回滚**

## 五、注意

- 10 个人同时提交时，任务会排队，因为 Runner 默认一次只跑一个（`C:\GitLab-Runner\config.toml` 里 `concurrent = 1`）。这正好体现真实团队的排队情况，想快一点可以改成 2。
- 所有 MR 都由你的账号创建，只有提交记录带着各开发人员的名字。
- GitLab 新账号可能需要先做身份验证，云端共享 Runner 才会跑 `sas_lint`。
- 你的 SAS 授权到 **2027 年 3 月 14 日**，过期后 `sas_run` 会全部失败，记得提前续期。
- 旧文件如果过不了 lint，可以修好，或把文件名写进 `ci/lint_exclude.txt`（`sas_run` 也会跳过它们）。
