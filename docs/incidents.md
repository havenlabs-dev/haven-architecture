# Incidents

Real postmortems from running this system. Each is written the way I want to
read one at 3 a.m.: symptom, what I believed, what was actually true, and the
rule it produced.

---

## The machine that was not down

**Symptom.** Mid-way through a multi-worker job the host stopped answering.
Everything timed out. It looked like the box had died under load — plausible,
given the hardware and the fact that I had just given it more work than usual.

**What I believed.** Thermal shutdown. I said so.

**What was actually true.** Home Wi-Fi had dropped, taking the mesh link with
it. The server had 141 days of uptime and was sitting at 42 °C throughout. It
never noticed.

**Why I got it wrong.** I reasoned from the load I had just added to the failure
I was seeing, and the VPN control plane still listed the node as `Online`, which
I read as "the machine is up but wedged."

**Rules.**
- A dead tunnel is not a dead machine. Confirm the host is down before naming a
  cause for why it went down.
- The mesh `Online` field can lie after the data path dies. `LastHandshake` and
  `uptime` do not.
- The heat concern was still legitimate — multi-worker jobs on this host now get
  their worker count named and approved before launch.

---

## The cron job that cooked the box

**Symptom.** Thermal alarm at 03:00. 39 °C to 77 °C, past the 75 °C trip.

**Cause.** A nightly job I had added ran a chess engine at full priority against
a backlog of games.

**The interesting part.** It was already running under `nice -n 19`. On an
otherwise idle single-tenant host, a niced process still receives the entire
CPU — scheduling priority only matters under contention. Niceness is not a
thermal control.

**Fix.** A guard that reads the sensor and pauses at 62 °C, resumes at 55 °C
(measured ~8 °C rise per game), plus a batch ceiling that refuses runs over 25
items and names the machine the work belongs on.

**Rule.** On a single-tenant box, the only protection against thermal load is
refusing the work.

---

## Four days of missing health data

**Symptom.** Health metrics stopped ingesting. The obvious suspect was expired
credentials with the upstream provider.

**Actual cause.** Database connection-pool poisoning. One failed connection
stayed in the pool and every subsequent borrower inherited the failure. The
integration was fine; the pool was not.

**Why it went unnoticed for four days.** Nothing watched for *absence*. There
was no error — the job ran, got nothing, and wrote nothing, successfully.

**Rules.**
- Monitor for silence, not just for errors. This produced the per-domain
  staleness detection described in [observability](observability.md).
- When one integration fails, check what it shares with the others before
  blaming the integration.

---

## The edits that kept vanishing

**Symptom.** Over weeks, edits to worker scripts would disappear across
restarts. Saved work would revert overnight.

**Cause.** Two bugs stacking. The process manager's shutdown routine unpacked a
`(pid, script)` tuple in the wrong order and unlinked the worker scripts
themselves. A separate monitor then noticed the missing files and restored them
with `git checkout`, reverting uncommitted work.

**Why it survived so long.** The second mechanism was helpful. It repaired the
damage well enough that the underlying deletion never surfaced as a failure.

**Rule.** Self-healing must be loud. An automated repair that hides the failure
it repairs removes the only available signal.

---

## The gate that failed closed on the write and open on the report

**Symptom.** A safety check reported success while doing nothing.

**Cause.** The write path failed and was caught; the reporting path did not know
that and returned its normal success value.

**Why it matters.** This turned out to be a class, not an instance. The same
shape appeared in a monitor that recorded "I asked about this" before attempting
delivery and swallowed the delivery error — permanently silencing the exact
condition it existed to catch. Its return value was honest while its stored
state lied.

**Rules.**
- Do the work, then record it. Never the reverse.
- A gate that fails closed on the write but open on the report is worse than no
  gate, because it manufactures confidence.

---

## Retiring an integration is not deleting the reads

**Symptom.** Months after a third-party integration was retired, a nightly job
was still writing records derived from it, and two analysis modules could not be
imported at all.

**Cause.** The retirement removed the reads and the UI. It did not enumerate the
**writers**, the **cron jobs**, or the **modules that imported its client
library**. Those modules raised `ModuleNotFoundError` on import — and their one
live caller caught the exception and returned an empty list, so a whole analysis
surface silently produced nothing for a month without a single error being
logged.

**Rules.**
- Retiring a dependency means enumerating its writers, its scheduled jobs, and
  its importers — not just its readers.
- `except Exception: return []` around an import is how a subsystem dies
  quietly. If a dependency is missing, that is worth an error.

---

## Absence is not failure

**Symptom.** A rebuilt analysis module was about to report that every tracked
behaviour had collapsed to zero.

**Cause.** The original implementation treated a missing record for a given day
as a recorded failure for that day. Logging had stopped for 12 days — not the
underlying activity, just the recording of it. Every rate computed over calendar
days therefore read as total collapse.

**Fix.** Compute rates only over days that actually contain records, and refuse
to draw conclusions at all when the most recent record is more than three days
old — deferring to the staleness detector that owns that question.

**Rule.** Missing data and zero data are different values. Systems that conflate
them generate confident, wrong, and occasionally cruel conclusions.

---

## Themes

Reading these together, three patterns account for nearly all of them:

1. **Declarations nothing enforces** — a policy with no chokepoint, a docstring
   with no test, a gate with no callers.
2. **Silence treated as success** — no error raised, so nothing was wrong.
3. **Reasoning from the change you just made** to the failure you are seeing,
   instead of from the evidence.
