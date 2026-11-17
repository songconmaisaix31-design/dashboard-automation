<div align="center">

# dashboard-automation

One GitHub Actions workflow that drives a dashboard refresh on a self-hosted Windows runner, so browser login state and local data never leave the company machine.

一个 GitHub Actions 工作流，把看板刷新调度到自托管的 Windows runner 上执行，浏览器登录态与本地数据都不离开公司机器。

[![Daily refresh](https://github.com/songconmaisaix31-design/dashboard-automation/actions/workflows/daily-refresh.yml/badge.svg)](https://github.com/songconmaisaix31-design/dashboard-automation/actions/workflows/daily-refresh.yml)
![Runner](https://img.shields.io/badge/runner-self--hosted%20Windows-blue)
![Schedule](https://img.shields.io/badge/schedule-daily%2006%3A00%20UTC-blue)
![Timeout](https://img.shields.io/badge/timeout-180%20min-blue)
![License](https://img.shields.io/badge/license-not%20specified-lightgrey)

</div>

> **How to read this page / 怎么读这页**
> English first, then the same content in Chinese — not a summary, not a separate half.
> 每段英文在前、同一内容的中文紧跟在后；不是摘要，也没有单独的「中文版」。

---

## At a glance · 一眼看懂

| Field · 字段 | Value · 内容 |
|---|---|
| **Problem<br/>问题** | The dashboard refresh needs a logged-in browser profile, DPAPI-protected credentials, and persistent local data — none of which can live on a hosted runner.<br/>看板刷新需要已登录的浏览器 profile、DPAPI 保护的凭据和持久化的本地数据，这些都没法放在托管 runner 上。 |
| **Fix<br/>做法** | A scheduled workflow that runs the existing local project through a self-hosted Windows runner, with dry-run and stage selection as dispatch inputs.<br/>一个定时工作流：在自托管 Windows runner 上调用已有的本地项目，支持 dry-run 与按阶段选择。 |
| **Cost<br/>代价** | A registered Windows runner with the runner labels below, a Python venv and the refresh script already on disk, plus repository variables and one secret.<br/>需要一台已注册、带下列标签的 Windows runner，磁盘上已有 Python venv 与刷新脚本，另需仓库变量与一个 secret。 |
| **Not<br/>不是** | Not the dashboard itself, and not runnable from this repository alone — the Python project it calls is not in this repo.<br/>不是看板本体，也无法只靠这个仓库跑起来：它调用的 Python 项目不在本仓库里。 |

## What is in this repository · 这个仓库里有什么

Two files. The control plane, nothing else.

两个文件。只有控制面，没有别的。

```text
dashboard-automation/
├── .github/workflows/daily-refresh.yml   # the whole control plane · 整个控制面
└── README.md
```

The refreshed project (`daily_refresh.py` and its `.venv`) lives on the runner at `DASHBOARD_RUNNER_ROOT`, not here. That split is deliberate: the workflow file can be reviewed and diffed in public while the data and credentials stay on the machine.

被刷新的项目（`daily_refresh.py` 及其 `.venv`）放在 runner 上的 `DASHBOARD_RUNNER_ROOT`，不在这里。这个分工是故意的：工作流文件可以在明面上审查和 diff，数据与凭据留在机器上。

## How a run works · 一次运行怎么走

The workflow is [daily-refresh.yml](.github/workflows/daily-refresh.yml). It has one job, `refresh`, with a 180-minute timeout, and two steps that actually do work.

工作流是 [daily-refresh.yml](.github/workflows/daily-refresh.yml)。只有一个 job `refresh`，超时 180 分钟，真正干活的只有两步。

| Step · 步骤 | What it does · 做什么 |
|---|---|
| Resolve target and run mode<br/>解析目标日期与运行模式 | Picks the business date, the stage list, and whether this run is dry-run / deploy. A scheduled run forces `dry_run=false` and `deploy=true`; a manual dispatch uses your inputs.<br/>确定业务日期、阶段列表，以及这次是 dry-run 还是 deploy。定时触发强制 `dry_run=false`、`deploy=true`；手动触发则用你填的入参。 |
| Run serial refresh<br/>串行执行刷新 | Fails fast if `DASHBOARD_RUNNER_ROOT` is unset, then calls the venv Python with `daily_refresh.py --date … --stages …`, appending `--model-profile`, `--dry-run`, `--deploy` as needed.<br/>若 `DASHBOARD_RUNNER_ROOT` 未配置则直接失败，然后用 venv 里的 Python 调用 `daily_refresh.py --date … --stages …`，按需追加 `--model-profile`、`--dry-run`、`--deploy`。 |

Two scheduling details are easy to miss and worth reading twice. The cron is `0 6 * * *` in **UTC**, which is 14:00 Asia/Shanghai; when no date input is given, the target date is computed as Asia/Shanghai "today minus one day" via the `China Standard Time` zone. And the concurrency group `dashboard-daily-refresh` uses `cancel-in-progress: false`, so runs queue instead of replacing each other.

两个调度细节容易看漏，值得读两遍。cron 是 **UTC** 的 `0 6 * * *`，即北京时间 14:00；不给日期入参时，目标日期按 `China Standard Time` 时区取「上海今天减一天」。并发组 `dashboard-daily-refresh` 用 `cancel-in-progress: false`，所以运行是排队，不是互相顶掉。

## Workflow inputs · 工作流入参

All six are `workflow_dispatch` inputs; a scheduled run ignores them and uses the forced defaults described above.

六个都是 `workflow_dispatch` 的入参；定时运行不看它们，用上面说的强制默认值。

| Input · 入参 | Default · 默认 | Meaning · 含义 |
|---|---|---|
| `date` | blank → Shanghai D-1<br/>留空 → 上海时区 D-1 | Business date as `YYYYMMDD`.<br/>业务日期，格式 `YYYYMMDD`。 |
| `stages` | `core,city_battle,package` | Comma-separated stages passed straight through to the script.<br/>逗号分隔的阶段名，原样传给脚本。 |
| `model_profile` | blank<br/>留空 | ID of a DPAPI model profile that already exists on the runner.<br/>runner 上已有的 DPAPI 模型 profile ID。 |
| `dry_run` | `true` | Plan only, no writes.<br/>只做计划，不落写操作。 |
| `deploy` | `false` | Deploy the API and web after the refresh.<br/>刷新后部署 API 与 web。 |
| `request_id` | blank<br/>留空 | Audit correlation ID from the dashboard; it only shows up in the run name.<br/>看板侧的审计关联 ID，只出现在运行名称里。 |

## Setup on the runner · 在 runner 上要准备什么

Before the first run, the runner needs all of these — the workflow fails loudly on the first two, and silently does nothing useful without the rest.

首次运行前 runner 上必须具备以下内容；前两项缺失时工作流会报错失败，其余缺失则不会报错但跑不出结果。

| Requirement · 要求 | Where it is referenced · 在哪里被引用 |
|---|---|
| Runner labels `self-hosted`, `Windows`, `X64`, `dashboard-runner`<br/>runner 标签 | `runs-on` |
| Repository variable `DASHBOARD_RUNNER_ROOT`<br/>仓库变量 | preflight check in step 2 · 第 2 步的前置检查 |
| `.venv\Scripts\python.exe` under that root<br/>该根目录下的 venv Python | preflight check in step 2 · 第 2 步的前置检查 |
| `daily_refresh.py` under that root<br/>该根目录下的刷新脚本 | preflight check in step 2 · 第 2 步的前置检查 |
| Repository variable `AUTOMATION_ENABLED` = `true`<br/>仓库变量 | job-level `if` — scheduled runs stay skipped until it is set; manual dispatch always runs.<br/>job 级 `if`：未设置时定时运行会被跳过；手动触发不受影响。 |
| Repository variable `VERCEL_BIN` (optional)<br/>仓库变量（可选） | step 2 env · 第 2 步环境变量 |
| Repository secret `VERCEL_TOKEN`<br/>仓库 secret | step 2 env, used by the deploy path while the scheduled run forces `deploy=true`.<br/>第 2 步环境变量；定时运行会强制 `deploy=true`，因此会走部署路径。 |

## Status and limits · 状态与边界

One commit (`5d04d90`, 2026-07-20), two files, no tests, no `LICENSE` file. The workflow has never been executed from a clone of this repository, because it cannot be: it needs the self-hosted runner and the local Python project.

一次提交（`5d04d90`，2026-07-20）、两个文件、没有测试、没有 `LICENSE` 文件。这个工作流从未、也无法仅凭本仓库的克隆被执行过：它需要那台自托管 runner 和本地 Python 项目。

Known limits, stated rather than hidden:

边界写出来，而不是藏起来：

- The refresh logic is not in this repository. Nothing here explains what a stage named `core` or `package` actually does; that lives in `daily_refresh.py`.<br/>刷新逻辑不在本仓库。这里没有任何内容说明 `core` 或 `package` 这类阶段到底做什么，那在 `daily_refresh.py` 里。
- Windows-only by construction: DPAPI profiles, `China Standard Time` zone lookup, and `.venv\Scripts\python.exe` paths.<br/>天然只支持 Windows：DPAPI profile、`China Standard Time` 时区查询，以及 `.venv\Scripts\python.exe` 路径。
- A scheduled run deploys for real (`deploy=true` forced), so the `AUTOMATION_ENABLED` switch is the only thing standing between a cron tick and a production deploy.<br/>定时运行会真的部署（强制 `deploy=true`），所以从一次 cron 触发到一次生产部署之间，唯一的开关就是 `AUTOMATION_ENABLED`。
- The badge above reflects the workflow file, not a passing run; a repository with no recent runs shows "no status".<br/>上面的徽章反映的是工作流文件存在，不代表最近有通过的运行；长期不跑会显示 "no status"。
- The `gh workflow run` equivalent of the manual dispatch was **not verified** here — only the Actions-tab path is confirmed by the trigger definitions in the YAML.<br/>手动触发对应的 `gh workflow run` 命令在本次**未验证**；只有 Actions 页面手动触发这条路径能由 YAML 里的触发定义确认。

## License · 许可

No `LICENSE` file is present, so no license is granted by default and reuse needs permission.

仓库里没有 `LICENSE` 文件，因此默认不授予任何许可，复用需要获得同意。

The workflow file is short enough to read as a reference for structuring your own self-hosted refresh job; copying it into your own repository is your call, and the absence of a license means you should ask first.

这个工作流文件足够短，可以当作「自己搭自托管刷新任务」的结构参考；是否拷进你自己的仓库由你决定，但没有许可意味着请先询问。
