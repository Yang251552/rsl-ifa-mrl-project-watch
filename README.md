# rsl-ifa-mrl-project-watch

GitHub Actions 每天抓 ETH 三个实验室（RSL、IfA、MRL）在招学生项目的 JSON feed，与各自的快照 diff，把结果写进 `state/<lab>/latest_run.json`，供本机每天 07:00 的 Claude Code 定时任务 `rsl-daily-md` 免鉴权读取（本机和云端都抓不动 sirop.org 的 Cloudflare 页面，公开 feed 由 Actions runner 抓）。2026-09-29 之前只监测 RSL（仓名 `rsl-project-watch`，RSL 状态在 `state/` 根下）。

同一个 job 依次检查三个实验室，状态互相独立（一个抓取失败不影响另两个）；下文字段与规则对三者都适用：

| 实验室 | 数据源 | 状态目录 | 新增汇总 |
|---|---|---|---|
| RSL（`python3 check_rsl_ifa_mrl_projects.py`） | [页面](https://rsl.ethz.ch/education-students/student-projects0/available-projects.html) 自带的 rssreader feed | `state/rsl/` | [`summary/rsl-new-projects.md`](summary/rsl-new-projects.md) |
| IfA（`… ifa`） | [页面](https://control.ee.ethz.ch/education/sa-ma-projects.html) 嵌入的 SiROP feed `7be87bdc…` | `state/ifa/` | [`summary/ifa-new-projects.md`](summary/ifa-new-projects.md) |
| MRL（`… mrl`） | [页面](https://mrl.ethz.ch/education/student-projects.html) 嵌入的 SiROP feed `c44dcc20…` | `state/mrl/` | [`summary/mrl-new-projects.md`](summary/mrl-new-projects.md) |

MRL 的 feed 里也有登记在 RSL 名下的联合指导题，所以同一个 url 可能同时出现在 RSL 和 MRL；读取方按 url 去重。SiROP 的发布日期已转成与 RSL 相同的 `DD.MM.YYYY`。

- 定时：`23 3,19 * * *`（UTC）= 瑞士 05:23 与 21:23（夏令时）/ 04:23 与 20:23（冬令时）；Actions 页也可随时手动 Run workflow。GitHub 的 `schedule` 只是尽力而为（无 SLA，高负载时延迟甚至丢弃）：2026-09-14/15 的 03:23 UTC 两次都晚了约 5.5 小时才创建运行，所以加一次前一晚的运行给 07:00 读取兜底；早上那次准点时数据更新鲜。
- 本地只跑自检 `python3 check_rsl_ifa_mrl_projects.py --selftest`；直接运行会改 `state/` 和 `summary/`，和远端快照打架。
- `state/<lab>/seen.json` 是快照 `{url: title}`；`state/<lab>/history.jsonl` 是 `baseline` / `new` / `removed` 事件流水（RSL 2026-09-13 之前的记录来自原 Mac 本地任务）。首次运行（快照不存在）记为 `BASELINE`，当时在挂的全部写成 `baseline` 事件，不算新增。
- `summary/<lab>-new-projects.md` 是历次新增汇总：每次检查后从该 lab 的 `history.jsonl` 全量重新生成，按检出日期（瑞士时间）分组、新的在上。

## `state/<lab>/latest_run.json`

    curl -fsS https://raw.githubusercontent.com/Yang251552/rsl-ifa-mrl-project-watch/main/state/rsl/latest_run.json

| 字段 | 含义 |
|---|---|
| `run_at` | 最近一次运行的时间，瑞士时区，带偏移 |
| `status` | 最近一次运行的结果：`NEW`（检出新增）/ `NONE` / `BASELINE`（快照重建）/ `FETCH_FAILED` |
| `listed` | 页面当前在挂项目数；`FETCH_FAILED` 时为 `null` |
| `new` | 最近 7 天检出的新增 `[{title, url, date, desc, push_date}]`，去重键是 `url`；超过 7 天的自动移除 |
| `error` | `FETCH_FAILED` 的报错，否则 `null` |

feed 为空按正常的 `NONE` 处理（`listed` 为 0，job 不报红），但快照保持不动：万一是 feed 端的软故障，恢复后不会把全部项目重报成新增。代价是空窗期内撤下的项目要等 feed 重新有内容时才记成 `removed`。

每条新增自带 `push_date`，即该由哪天 07:00 的推送列出：06:40 之前检出的算当天，之后的算次日（留 20 分钟给运行、push 和 raw 的 5 分钟缓存）。每条在 `new` 里保留 7 天，所以 cron 延迟、任意时间手动触发都不会让它在该推的那天之前被冲掉。推送只列当天的；本地任务哪天没跑，那天的条目不补推，但一定在对应的 `summary/<lab>-new-projects.md` 里。前提：读取不早于 06:50。
