# Observability

## Metrics

Netdata runs in a container, bound to the tailnet interface, giving per-second
system metrics: CPU, memory, disk I/O, thermals, per-process resource use. On a
host this small, the thermal and memory graphs are the two that actually change
decisions.

Application state lives in the database rather than a separate metrics store —
226 tables, so "is the ingest healthy?" is a query, not a dashboard integration.
That is a deliberate simplification for a single-host system: one datastore to
back up, one to restore, one place to look.

## Alerting: edge-triggered or not at all

The most important rule in this layer was learned by getting it wrong.

An alert that fires every time a condition is true will fire every night that
the condition stays true. Within a week it is noise, and noise is worse than
silence because it trains you to swipe away the one that matters. A backup
monitor here did exactly that, and a notification layer did it at scale — 37
notifications in a single morning, at which point none of them were read.

Everything now fires on **state transition**:

- An alert opens an *episode* when a metric crosses into a bad band, and stays
  open. It does not re-fire while the state persists.
- It closes when the metric returns, and only then can it fire again.
- Alerts that have never been acted on are muted automatically, with the
  numbers attached, and a protected set can never be muted.

A related failure: something that asks a question, records that it asked, and
never actually delivers the question. A monitor here committed "I asked about
this" to the database *before* attempting delivery, and swallowed delivery
errors. A domain whose alert never reached me was marked as asked and went
permanently quiet — the exact condition the check existed to catch. Its return
value was honest while its stored state lied.

**Deliver first, record second.** A failed delivery should retry, not
self-silence.

## Integrity checks that are allowed to fail

Derived numbers drift from their sources. Nine nightly checks assert properties
that were each violated at some point in production — transfers must not appear
as revenue, hand-entered records must not double-count the automated feed, and
so on.

The rule that makes them worth anything:

> **A check that cannot fail is not a check. Fault-inject it or it is decoration.**

Every check ships with a test that reintroduces the original bug on a throwaway
copy of the database and asserts the check goes red. This is not ceremony — it
immediately caught a bad check of my own. My first version of "transfers must
not be counted as revenue" asked the same helper function that did the
classifying whether it had found the transfers. Disabling the classifier made
the check and the buggy code agree, and the check stayed green with the bug
live. It now validates against an independent first-party source instead.

**A check that shares a primitive with the thing it checks is measuring its own
opinion.**

The same rule applies to the checks themselves: the integrity modules and their
tests are on a protected-paths list, so an automated fix is forced to produce a
diff against the logic rather than a quieter check.

## Knowing when a data source has gone quiet

Silence is the failure mode that monitoring usually misses, because nothing
errors — a feed simply stops and everything downstream keeps serving stale
numbers confidently.

Each ingest domain has its own cadence learned from its own history: the median
gap between active days over the last 40 active days, with a floor so that a
naturally sparse source is not accused of being broken at its normal rhythm.
Three times its own rhythm, never below the floor, and it asks once — then stays
quiet until the source moves again.

This caught a real one while this repository was being written: a data source
had been silent for 12 days, and a downstream analysis was still willing to draw
conclusions from a window that had closed two weeks earlier.

## The failure class this all defends against

Nearly every incident in [the log](incidents.md) is the same shape:

> **A declaration that nothing enforces.**

A policy table whose callers ignore it. A docstring claiming behaviour the code
does not have. A gate function with zero call sites. A capability description
that gets repeated as fact.

The countermeasure is always the same: one chokepoint that everything must pass
through, plus a conformance test that fails when something bypasses it.
