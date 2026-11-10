# inet4031-week10 (Week 10 Lab Reference Repo)
 
Week 10 of INET 4031: end-to-end performance. Students load their incident tracker with 10,000 synthetic tickets and watch the same code slow down. They measure it from the outside with k6, then find where the time actually goes: Postgres answers in a few milliseconds, and the web pod spends the rest building a multi-megabyte page. They fix that in the app, learn what a missing index looks like on a different query, and then meet a second problem from the infrastructure side, a CPU limit lowered by hand that throttles the web pod. They fix it through OpenTofu, add a k6 threshold gate to GitHub Actions, and write a two-incident runbook with their measured numbers.
 
## Where the lab is
 
The step-by-step instructions are in this repo's [wiki](https://github.com/INET4031-Labs/inet4031-week10/wiki). Students follow the wiki, not this README.
 
## Nothing to clone this week
 
As in Weeks 5 to 9, there are no starter files. Students create every file themselves from the wiki, inside the personal repo they have used all semester. They do not clone this repo.
 
## What students end up with
 
In their own repo, committed through pull requests:
 
```
.github/
  workflows/
    load-test.yml     (new: a k6 smoke test that runs on pull requests that touch the app)
week-4/
  app/app.py          (edited: the home page query gets LIMIT 50)
  app/templates/index.html   (edited: the heading says "Latest tickets")
week-5/
  db.tf               (edited: Postgres loads pg_stat_statements at startup)
  web.tf              (edited: the Deployment runs the week-4-web:v2 image)
week-10/
  sql.sh              (run SQL in the database pod, or count rows in every table)
  run-k6.sh           (run a k6 test against the web Service, save it, print a summary)
  k6-stat.sh          (read one number from a saved run)
  throttle.sh         (ask Prometheus how often the web container was throttled)
  smoke-test.js       (3 users for a minute, with thresholds)
  ramped-test.js      (up to 20 users over two minutes)
  check.sh            (PASS or FAIL for the whole setup, nine checks)
  README.md           (what this does, two runbooks with measured numbers)
```
 
Outside the repo:
 
```
the VM          k6 in /usr/local/bin, installed from a pinned release and checked against its SHA-256
                saved results: ~/.k6-<label>.json and ~/.throttle-<label>
the cluster     a web:v2 image imported into k3s by hand, and, for a few minutes in Part 7,
                a CPU limit changed with kubectl and then put back by tofu apply
the database    the pg_stat_statements extension and an index, idx_tickets_status_created,
                both created by hand, plus 10,000 synthetic tickets that Part 9 deletes
```
 
## Prerequisites
 
- Week 8 finished: the hardened `web` with its 256Mi memory limit and 500m CPU limit, the ruleset, the gate, and the deploy script. Week 9 is not required, and no step here depends on it. If its backup cron job is running, its `pg_dump` queries show up in `pg_stat_statements`, and the wiki tells students to ignore them.
- Week 6's Prometheus running, with the `kube-state-metrics` and `node-exporter` pods and the `week-6/pq.sh` script. The wiki leaves a port-forward to Prometheus running in its own terminal.
- The Week 4 starter app unchanged, or changed only in ways that keep its query line `SELECT * FROM tickets ORDER BY id DESC`. The wiki's fix commands match that exact text.
- The `week-4-web:latest` image still in k3s, and Docker on the VM to build `v2`. The wiki checks for the image.
- Pods labeled `app=web` and `app=db`. The number of `web` pods varies between students, because it depends on what they last set in Weeks 6 and 7. The wiki's Prometheus queries look at the busiest pod, so one pod or several both work. A cluster built from the Week 4 kompose manifests carries `io.kompose.service` labels instead and will not match its OpenTofu files.
- A VM disk under about 80% full. The wiki ends with a section on checking and extending the disk.
- The GitHub CLI signed in as the repo's owner, with permission to push workflow files, as in Week 6.
- `sudo` on the VM, to install k6 and to import the image.
## What the lab covers
 
- End-to-end latency, percentiles, and why p95 is the number to promise
- The USE method (utilization, saturation, errors) read through Prometheus
- Installing a tool from a pinned release and verifying its checksum
- k6: virtual users, stages, thresholds, and a summary export that scripts can read
- Enabling `pg_stat_statements` through the pipeline, and reading rows per call next to mean time
- `EXPLAIN (ANALYZE, BUFFERS)`: an index scan that returns every row, then a sequential scan against an index scan on a filtered query
- Building, tagging and importing a new image tag, and pointing OpenTofu at it
- CPU limits and CFS throttling, and telling cause from symptom
- Drift: a change made by hand, found by `tofu plan`, fixed by `tofu apply`
- A GitHub Actions job that builds a Compose stack on the runner, seeds data, and runs k6 against it
- What a performance gate can and cannot catch, and whether it should be required
- A runbook with measured before and after numbers
## Things to know before running it
 
- **The database incident is not a missing index.** The starter app's only list page runs `SELECT * FROM tickets ORDER BY id DESC` and renders every row. That query already uses the primary key, so an index cannot help the page. At 10,000 rows the page is about 7 MB, Postgres takes about 5 ms, and the web pod spends the rest building HTML. The fix is `LIMIT 50` in the app. The reference lab's index exercise is kept as Part 6, on a status-filter query the app does not run, and the wiki says so.
- **10,000 rows, 3 users.** The Week 8 web memory limit is 256Mi. Measured on this app, one request for 50,000 tickets needs about 200 MiB and five concurrent requests about 700 MiB, so the pod would be OOM-killed. At 10,000 rows five concurrent requests peak near 207 MiB.
- **Expected numbers, measured on a sandbox with 3 VUs:** a few tickets give a p95 of roughly 25 to 50 ms. With 10,000 more, p95 is about 950 to 1,000 ms against the 500 ms threshold. After the fix the page is about 38 KB and p95 is back near the baseline. The ramped test (20 users) gives a p95 of about 76 ms and about 17 requests per second. A slower VM will show larger numbers.
- **Throttling numbers are an emulation, and at a different limit than the wiki uses.** The wiki lowers the limit to 50m. I could not create a real CFS limit, so I emulated 100m by holding the app to a 10% duty cycle. The ramped test went from p95 76 ms to 1,432 ms and from about 17.6 to 8.1 requests per second, and failed its threshold. Treat those figures as the direction of the effect. 50m is a tighter limit, so the real symptoms should be stronger, and I did not test it.
- **Throttling also appears in Part 4,** at the correct 500m limit, because each request for the 10,000-row page is a burst of about a third of a second of CPU. That is deliberate. The limit is enforced every 100 ms, so a pod can be throttled in most periods while its average CPU use over three minutes stays well under 0.5 cores, and the wiki explains this. In Part 7 the CPU use sits at the 50m ceiling instead. Q8 asks students to tell throttling as a cause from throttling as a symptom by comparing CPU use and throughput with the limit.
- **k6 runs on the VM and aims at the `web` Service's ClusterIP,** not at a port-forward. A port-forward is one connection to one pod and would not measure the Service. k6 shares the VM's CPU with the cluster. Part 1 says so, but no discussion question asks about it.
- **The Prometheus scrape interval is one minute.** The wiki has students run two-minute tests and wait about a minute before querying. The throttling ratio is `throttled periods / periods` over three minutes, so idle time does not dilute it.
- **`kubectl set resources` is surgical on purpose.** It lowers one limit and nothing else. Week 9's problems showed what a full `kubectl apply -f` of the Week 4 manifests does to a cluster. The deploy loop only runs on new commits, so it does not notice the drift. `tofu plan` does.
- **The fix commands are safe to run twice.** The `LIMIT 50` edit is anchored on the closing quote of the query string. An earlier version was not, and a second run stacked `LIMIT 50 LIMIT 50`. The `db.tf`, CPU limit and image tag edits are also safe to repeat.
- **Every OpenTofu change here is an in-place update,** so the Week 8 approval step never fires. The wiki's plan checks expect exactly one `will be updated in-place` line each time.
- **The CI gate builds the Week 4 Compose stack** on `ubuntu-latest` with the placeholder values from `.env.example`, waits for both healthchecks, seeds 10,000 tickets, and runs the same `smoke-test.js` for 30 seconds. k6 is pinned by version and SHA-256. The gate has a `paths` filter and is not on the ruleset's required list. A flaky performance check must not be able to lock students out of merging. Q10 asks whether it should be required.
- **The `v2` image is built and imported by hand,** and the pull request is only safe to merge after the import. The wiki says so before the pull request step.
- The app's own `/health` route is what the wiki uses to check that the VM can reach the Service.
## Common failure points
 
- **The `curl` health check hangs or fails.** The VM cannot reach the `web` Service's ClusterIP. k6, `run-k6.sh` and `check.sh` all depend on it. Check `kubectl get service web` and `kubectl get endpoints web`.
- **The k6 checksum line says `FAILED`.** The download is damaged or the CPU type is not amd64 or arm64. Run the block again.
- **`SHOW shared_preload_libraries` prints nothing after the first pull request.** The `db` pod has not restarted with the new argument yet. Check `kubectl get pods` and `tail ~/deploy.log`.
- **The plan after the `db.tf` edit shows more than one changed resource, or a `must be replaced` line.** The cluster has drifted from the files, usually from a `kubectl apply` of the Week 4 manifests. The label check in Before You Begin catches it early.
- **`git diff --stat` after the fix commands lists nothing.** The student's query line differs from the starter's. The wiki tells them to add `LIMIT 50` by hand.
- **`web` is in `ErrImagePull` after the `bound-the-query` pull request merges.** The pull request merged before the `v2` image was imported into k3s. Import it with the commands in Part 5 Step 2.
- **`throttle.sh` says there is no data.** Prometheus has not scraped since the test ended, or the container CPU-period series are missing. Wait a minute. If it persists, query `container_cpu_cfs_periods_total{container="web"}` directly. Those series exist only for containers with a CPU limit, which Week 8 added.
- **`gh pr checks` prints `no checks reported` and stops.** It ran before GitHub had started the checks, which does not mean they passed. The wiki puts a `sleep 15` before each `gh pr checks`. If the message still appears, run the command again.
- **The Part 4 CPU answer is well under 0.5 while the throttled share is high.** That is expected. See the Part 4 bullet above.
- **`pq.sh` cannot reach Prometheus.** The port-forward from the wiki is not running in its terminal.
- **The seeded or ramped test never finishes or the VM becomes sluggish.** The load generator shares the VM's CPU. Keep to the wiki's user counts.
- **The `performance smoke test` check passes on the attack pull request.** The `LIMIT` was not actually removed, so check the `sed` output and `git diff --stat`.
- **The `performance smoke test` check fails on a good pull request.** The runner was slow. The threshold is 500 ms at 3 users against a 38 KB page, so this should be rare. Re-run the job, and tell the instructor if it repeats.
- **`check.sh` fails the drift check.** `tofu plan` found changes. The usual cause is the Part 7 CPU change not yet reverted with `tofu apply`.
- **`check.sh` fails the `restored` check.** The ramped test labeled `restored` was never run, or ran with the limit still at 50m.
- **The runbook still has `_HERE` in it.** One of the seven saved files was missing when the `sed` ran. The wiki lists them first.
- **Pods sit in `Pending`.** Check `df -h /` first. The wiki ends with a section on disk pressure.
## Further reading
 
- k6 thresholds: https://grafana.com/docs/k6/latest/using-k6/thresholds/
- The USE method, by Brendan Gregg: https://www.brendangregg.com/usemethod.html
- PostgreSQL `pg_stat_statements`: https://www.postgresql.org/docs/16/pgstatstatements.html
- PostgreSQL, using `EXPLAIN`: https://www.postgresql.org/docs/16/using-explain.html
- Kubernetes, assigning CPU resources to containers: https://kubernetes.io/docs/tasks/configure-pod-container/assign-cpu-resource/
- GitHub Actions, the `pull_request` event and path filters: https://docs.github.com/actions/writing-workflows/choosing-when-your-workflow-runs/events-that-trigger-workflows#pull_request
