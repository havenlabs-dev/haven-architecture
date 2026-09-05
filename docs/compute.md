# Compute

## The machine

| | |
|---|---|
| Chassis | Consumer laptop, ~2016 |
| CPU | AMD A12-9720P — 2 physical cores, 4 threads, 2.7 GHz max |
| RAM | 11 GiB usable, 4 GiB swap |
| GPU | None usable for compute |
| Thermal | Idles ~40 °C; alarm threshold set at 70 °C, hard trip at 75 °C |
| Uptime | 147 days |

It is a laptop on a shelf. That is not an accident of poverty so much as a
deliberate constraint: a weak host forces every design decision to be honest
about cost. Nothing gets added because it "might be useful."

## Why the constraint is load-bearing

With 11 GiB of RAM and four threads, the resident set is a budget, not an
afterthought. The current tenants:

- FastAPI / uvicorn (the API and its workers)
- Next.js dashboard
- n8n + Postgres (automation)
- Netdata (metrics)
- SearXNG (private metasearch)
- TeslaMate + Postgres (vehicle telemetry)

That leaves roughly 8 GiB available in steady state. Any proposal to add a
resident service starts by checking `free -h` and current load, and by naming
what it will cost when idle — not when busy.

## The rule that follows: heavy work runs elsewhere

Model inference, embedding runs, and bulk indexing are **not scheduled on this
host**. They are refused. Writing that code here is fine; running it is not.
Batch jobs of that shape are placed on a separate machine with a real GPU.

This came from an incident rather than a principle. A nightly job I added ran a
chess engine at full priority against a backlog of games. The host went from
39 °C to 77 °C and tripped its own thermal alarm at 03:00. `nice` was not the
fix — on an otherwise idle box, a niced process still gets the whole CPU. The
fix was a guard that reads the thermal sensor and pauses at 62 °C, resuming at
55 °C, plus a batch ceiling that refuses runs over 25 items and names the
machine the work belongs on.

```
PAUSE_C  = 62     # measured ~8 °C rise per game
RESUME_C = 55
BATCH_GUARD = 25  # refuses larger runs and names the right host
```

The general lesson: **on a single-tenant box, scheduling priority does not
protect you from thermal load.** Only refusing the work does.

## Capacity discipline in practice

- Check `free -h` and load average before adding anything resident.
- Prefer a cron job that exits over a daemon that waits.
- Prefer a package over a container; prefer a container over a VM.
- Swap exists (4 GiB) and is used lightly. Sustained swap is treated as a
  design failure, not a capacity solution.
- The database is 39 MB. Keeping it small is a feature: it makes the nightly
  `sqlite3 .backup` fast enough to be non-negotiable.

## Where this goes next

The plan is to move the heavier half onto a second machine and keep this host as
the always-on service tier. The split is by workload class — persistent services
here, burst compute there — rather than a general migration. Until that lands,
the refusal rule above is what keeps the box alive.
