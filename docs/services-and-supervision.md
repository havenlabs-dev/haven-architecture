# Services and supervision

## Layout

Two supervision systems, chosen per workload rather than by preference.

**systemd** owns anything that is part of the application:

| Unit | Role |
|---|---|
| `haven-dev.service` | The backend. Owns a process manager which in turn owns the API and every worker. `Restart=always`. |
| `haven-web.service` | The Next.js dashboard. `Restart=always`. |
| `tailscaled.service` | Mesh VPN node agent. |
| `docker.service` | Container runtime. |
| `cloudflared-n8n.service` | Single scoped tunnel for one automation webhook. |

**Docker** owns third-party services with awkward dependency trees, where a
container is genuinely simpler than packaging: automation engine and its
Postgres, metrics, private metasearch, vehicle telemetry and its Postgres.

The split is a judgement, not dogma. Anything I have to debug at 2 a.m. runs as
a systemd unit where `journalctl` and `systemctl status` work normally.

## One owner per process

The API is **never** started or stopped by hand. It is started by a process
manager, which is started by systemd. The restart procedure is to kill the
manager and let `Restart=always` bring the whole tree back:

```bash
kill $(systemctl show haven-dev --property=MainPID --value)
```

This is a rule with a scar behind it. Manually starting the API produced
duplicate uvicorn instances bound to the same port — two processes, one socket,
requests landing unpredictably in either, and state diverging between them. The
symptom was intermittent and looked like a bug in the application.

**One process, one owner.** If a supervisor is responsible for a service, no
human starts that service.

## The process manager, and a bug worth remembering

The manager owns worker processes and restarts them when they die. For weeks
there was a mystery: edits to worker scripts would vanish across restarts. Code
that was definitely saved would be back to an earlier version the next morning.

Root cause was two bugs stacking:

1. The manager's shutdown routine unpacked a `(pid, script)` tuple in the wrong
   order and called `unlink()` on what it thought was a temp path — it was
   deleting the worker scripts themselves.
2. A separate monitor noticed the missing files and restored them with
   `git checkout`, silently reverting uncommitted work.

Neither alone was visible. Together they formed a loop that destroyed edits and
covered its tracks. The second bug was "helpful," which is what made it
dangerous — a repair mechanism that hides the failure it is repairing removes
the only signal that something is wrong.

**Rule that came out of it:** a self-healing action must be logged loudly. If
the system fixes something on its own, that is an event to report, not a
detail to swallow.

## Scheduled work

44 cron entries, roughly in three classes:

- **Ingest** — pull from wearables, banking, calendar, vehicle telemetry.
- **Integrity** — nightly checks that the derived numbers still agree with their
  sources, and that the things which should have run, ran.
- **Housekeeping** — backups, retention, log rotation, alarm sweeps.

Conventions, all of them earned:

- Every job sources its environment explicitly (`set -a && . ./.env && set +a`).
  A cron job does not inherit an interactive shell. A script that silently
  authenticated as nobody and returned a clean empty result cost a day of
  debugging once.
- Every job runs under the project virtualenv by absolute path. The system
  interpreter does not have the dependencies, and a job that half-works is
  worse than one that fails loudly.
- Every job appends to its own log under `/tmp`. A job with no log is a job you
  cannot diagnose.
- Long or CPU-bound jobs run under `nice -n 19` **and** a thermal guard — see
  [compute](compute.md) for why `nice` alone is insufficient.
