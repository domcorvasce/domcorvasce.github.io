---
title: "How to repair Databricks jobs starting from a specific task"
date: 2023-07-01T14:22:35+02:00
draft: false
---

> Is it possible to repair a Databricks job starting from a specific task?

This question was asked in one of the company's Slack channels a couple of days ago. Heck! I must have asked myself this question a dozen of times in the last year.

Let's get right to it: as of 2023, Databricks UI does **not** allow you to select which tasks to re-run
when you repair a multi-task job. Either you trigger a new job run or you repair the failed one by  [re-running the failed and skipped tasks](https://docs.databricks.com/workflows/jobs/repair-job-failures.html#re-run-failed-and-skipped-tasks).

The catch is that the [**Databricks Jobs API 2.1**](https://docs.databricks.com/api/workspace/jobs) actually implements a field to [specify which tasks to re-run](https://docs.databricks.com/api/workspace/jobs/repairrun) during a job repair. The `rerun_tasks` field specifies the names of the tasks we want to re-run.
It does not matter whether the tasks were successful or not during the last job run.

We can use the [Databricks CLI](https://github.com/databricks/cli/blob/main/docs/commands.md#databricks-jobs-repair-run---repair-a-job-run) to trigger a job repair that uses `rerun_tasks`. There exist two Databricks CLIs. The [legacy](https://docs.databricks.com/archive/dev-tools/cli/index.html) one written in Python, and the [stable](https://docs.databricks.com/dev-tools/cli/databricks-cli.html) one written in Golang. This article assumes you're using the latter (version 0.200 or higher).

```shell
databricks jobs repair-run --json '{"run_id": JOB_RUN_ID, "rerun_tasks": ["task_B", "task_C"]}'
```

We can also trigger the job repair procedure exposed via Databricks UI:

```shell
databricks jobs repair-run --json '{"run_id": JOB_RUN_ID, "rerun_all_failed_tasks": true}'
```

You cannot have `rerun_tasks` and `rerun_all_failed_tasks` in the same request or Databricks API will complain.