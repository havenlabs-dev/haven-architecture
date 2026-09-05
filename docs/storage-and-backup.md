# Storage and backup

## Storage

A single 931 GB SATA SSD, ext4, 9% used. One filesystem, no LVM, no RAID.

**There is no RAID, and that is a choice with a cost.** On a single-disk host,
a drive failure is total loss of local state. The mitigation is not redundancy
in the box — it is that the data is small, the backups are frequent, and at
least one copy lives on different hardware. Mirroring would protect against disk
failure but not against the host being stolen, dropped, or cooked; off-box
copies protect against all four. Given one machine and a 39 MB database, off-box
frequency buys more than a second spindle would.

Stated plainly so nobody has to guess: **this system is not highly available.**
It is a single host. What it has is a short, tested path back.

## The database

SQLite, 39 MB, 226 tables, **WAL mode**.

WAL matters operationally: readers do not block the writer, so the nightly
backup and the integrity checks can run against a live database without
stalling the API. It also means a naive `cp` of the `.db` file is not a valid
backup — the write-ahead log may hold committed data that the main file does
not. Every backup path uses `sqlite3 .backup`, which is WAL-aware and produces a
consistent snapshot of a running database.

```bash
sqlite3 "$SRC" ".backup '$tmp'" && gzip -f "$tmp"
```

## Three layers

| Layer | What | When | Retention |
|---|---|---|---|
| 1 | `sqlite3 .backup` + gzip of the database | 03:00 daily | 14 days |
| 2 | State archive — config, keys, service state | 03:05 daily | rolling, ~43 MB/night |
| 3 | Off-box: laptop pulls a copy; bare git mirror of the server repo | after commits / nightly | full history |

Layer 3 is the one that matters. Layers 1 and 2 live on the same disk as the
thing they protect, which makes them useful for "I deleted the wrong row at
2 a.m." and useless for drive failure.

The code mirror is deliberately **not** on a public host. The repository history
predates the current secret hygiene and contains credentials that have since
been rotated and revoked — but rotated is not the same as absent, so the
replication path is a bare repo on the server and from there to an off-box
clone, and never to a public forge. Scrubbing that history is a separate,
tracked task.

This is also why the repository you are reading was authored from scratch rather
than filtered out of the working tree. Publishing a subset of a repository whose
history contains secrets is a filtering exercise you only have to get wrong
once; writing fresh files has no such failure mode.

## The backup alarm, and why it was wrong at first

A monitor checks nightly that backups actually ran, and pages if they did not.
The first version cried wolf: it alerted on a condition that was routinely true
and harmless, so the alert stopped carrying information. An alarm that fires
when nothing is wrong trains you to ignore it, which is worse than no alarm.

It was rewritten to be **edge-triggered** — it fires on the transition into a
bad state, not on every night the state is bad. Same rule now applies across the
monitoring layer; see [observability](docs/observability.md).

## Restore drill

Restoring is deliberately three commands, because a restore path you have not
run is a hypothesis:

```bash
# 1. stop the writer  (systemd restarts the manager; never kill uvicorn directly)
kill $(systemctl show haven-dev --property=MainPID --value)

# 2. restore the snapshot
gunzip -c /home/aknn/backups/haven-db-YYYY-MM-DD.db.gz > /home/aknn/haven/data/haven.db

# 3. verify before letting anything write to it
sqlite3 /home/aknn/haven/data/haven.db 'PRAGMA integrity_check;'
```

Step 3 is not optional. Restoring a corrupt snapshot on top of a working
database converts a recoverable incident into an unrecoverable one.

## Practices that came from mistakes

- **Snapshot before destructive operations.** Cascading deletes record a
  restorable snapshot of everything they touch. This turned "should I gate
  deletes behind an approval prompt?" into "deletes are reversible, so they do
  not need a prompt" — the better answer, because it removes the friction
  instead of adding a checkbox.
- **Never trust a docstring.** A function documented as deduplicating records
  was in fact summing them, and had been for months. Nobody had tested the
  claim. Documentation is not a control.
