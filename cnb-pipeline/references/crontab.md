# CNB crontab schedules

> An on-demand reference of `cnb-pipeline`: **crontab schedule config and sync behavior**. Read when writing scheduled-task config, or when a user asks why a scheduled task "didn't take effect".

## Mental model (establish before diagnosing)

`.cnb.yml` is the config source, but the cloud-native build's **scheduled-task list is a separate persisted dataset**; the two are aligned by sync:

- A push **that includes a `.cnb.yml` change** → the platform parses and syncs that branch's scheduled tasks (adds/deletes are persisted automatically)
- **In all other cases nothing syncs automatically** — this is the root cause of "configured but not registered"

So "the crontab is written in `.cnb.yml`" ≠ "the scheduled task is active". When diagnosing, always first check whether the two datasets are aligned.

## Config essentials

```yaml
main:                              # must be one explicit branch name
  "crontab: 30 5,17 * * *":        # "crontab: " prefix + POSIX cron expression
    - name: nightly-build-and-test
      stages:
        - name: run-tests
          script: echo "Running scheduled tasks..."
```

- **Branch must be explicit**: no glob (e.g. `feature/*`) and no `$` fallback branch; scheduled tasks hang under a concrete branch name
- **Timezone**: cron evaluates in the system timezone (`Asia/Shanghai`), not UTC
- **Minimum interval is 5 minutes**: shorter expressions (e.g. `* * * * *`) cannot be submitted
- **Execution identity = last modifier**: the pipeline runs as the user who last added/modified the crontab config and pushed; `CNB_TOKEN` is scoped to the current repo by default
- **Cross-repo needs**: within a root org, an admin must enable "allow scheduled tasks to operate across repos" in "Org settings → cloud-native build controls"; across root orgs, use API triggering (`api_trigger`)

## Sync rules quick table

| Scenario | Auto-sync? |
|---|---|
| Push that includes a `.cnb.yml` change (including first push of a new branch) | Yes |
| Only a same-repo template file changed (e.g. `crontab.yml`), `.cnb.yml` untouched | **No** |
| Template lives in another repo and that repo updated | **No** (template repo updates do not notify referencing repos) |
| Repo unarchived and restored | **No** — click sync or re-push the branch |

> Note: with `include` templates, scheduled tasks may live in the **template file**; if they don't show in the Flow view, read the template source.
> After sync, the new task's owner is the syncing user, not the original config author.

To re-align manually: in the repo web UI, "cloud-native build → scheduled tasks" (path `/{slug}/-/build/crontab`), pick the branch, click **sync**. Field details: fetch the official crontab doc per `doc-index.md`.

## Deletion and cleanup

- **Delete one task**: remove the `"crontab: ..."` key-value pair from that branch's config and push (triggers auto-sync)
- **Delete all of a branch**: remove all crontab entries under that branch
- **Auto cleanup**: tasks are cleaned automatically when the branch is deleted, the repo is deleted, or the repo is archived
