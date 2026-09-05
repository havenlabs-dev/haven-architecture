# Operations

## Restart

```bash
# The only correct restart. systemd (Restart=always) brings the tree back.
kill $(systemctl show haven-dev --property=MainPID --value)
```

Never start the API by hand — see [services](services-and-supervision.md) for
the duplicate-instance incident that produced this rule.

**Before restarting, check three things:**

1. Is a long-running job in flight? Killing the manager kills its children.
2. Is anyone actively using the system? A restart mid-conversation is a visible
   failure, not an invisible one.
3. Did the last deploy commit? An uncommitted worker edit can be reverted by
   the restore monitor.

## Log locations

| Log | Contents |
|---|---|
| `/tmp/haven-api.log` | API request log and tracebacks |
| `/tmp/haven-backup.log` | Nightly database backup |
| `/tmp/haven-backup-check.log` | Backup verification sweep |
| `journalctl -u haven-dev -f` | Service-level start/stop/crash |
| `docker logs <name>` | Per-container |

## Diagnosis order

Learned by getting it wrong in this order:

1. **Is the host actually up?** `uptime`, thermals, and the VPN handshake
   timestamp. A dead network path looks exactly like a dead machine from the
   outside, and the mesh control plane will still say `Online`. Verify before
   theorising.
2. **Is the process running, and is there exactly one of it?** `systemctl
   status`, then `ss -tlnp` on the port. Two listeners on one port produce
   symptoms that look like application bugs.
3. **Read the log before reproducing.** The system records its own diagnosis;
   re-deriving it by hand wastes the evidence.
4. **Check the data before the code.** Most "the app is broken" reports here
   have turned out to be a stalled ingest or a client-side fetch that failed
   once and never retried — not a backend fault.
5. **Only then change something.**

## Client-side failure has its own signature

Twice, "the app isn't working" was a client bug with a healthy backend. The
tell: notifications kept arriving (they poll on an interval) while every other
panel was blank (single fetch on mount, no timeout, no retry, error swallowed by
`.catch(() => {})`).

One failed request left a panel permanently empty until the app was force-quit.
The fix was a shared fetch helper with a timeout and bounded retry — **GET only,
because retrying a POST can duplicate a write**, and a silent duplicate record
is worse than a failed one. A failed panel and an empty panel must not look the
same.

## On-call reality

Single operator, no rotation. That shapes the design more than any preference:

- Failures must be **loud at the moment they happen**, because there is no
  second person to notice later.
- Alerts must be **rare enough to still be read** — hence edge-triggering.
- Recovery must be **short enough to do half-awake** — hence a three-command
  restore.
- Anything automated must **report what it did**, because an unattended fix that
  hides the failure removes the only signal available.
