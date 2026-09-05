# AI-assisted development

This system is built with heavy AI assistance, and saying so is more useful than
pretending otherwise. What follows is the working method, including the parts
that failed.

## The honest framing

I am not a full-time software engineer. HAVEN is ~104k lines of Python across
404 modules, and I could not have written that volume by hand. What I do own is
the architecture, the operational decisions, the constraints, and — critically —
the verification. The model writes most of the code. It does not decide what is
true.

That division only works if verification is real, which is where most of the
engineering effort actually goes.

## The guardrails

**Fault injection over assertion.** Every integrity check ships with a test that
reintroduces the original bug and asserts the check goes red. This caught a
check that an AI (and I) had written incorrectly — it validated its own opinion
rather than the data, and stayed green with the bug live. A test suite that only
proves the happy path is a test suite that agrees with whoever wrote it.

**Verify against production data, not the description.** The rebuilt analysis
module described in [incidents](incidents.md) passed its unit tests and would
have shipped nonsense, because the tests encoded the same wrong assumption as
the code. Running it against real history is what exposed it — twice: once for
treating missing records as failures, once for flagging fourteen items where
only four were real.

**Negative results get published.** The same module looks for correlations
between behaviour and health metrics. The strongest relationship in ninety days
of data explains about 8% of the variance, and the "large" effect it appeared to
show rested on three days of data on one side of the comparison. The threshold
stayed where it was and the module reports that nothing cleared it. Lowering a
threshold until the output is non-empty is the most common way an analysis
becomes a lie.

**Protected paths.** Auditing modules and their tests are on a list that
automated changes cannot touch. A failing check must produce a diff against the
logic, never a quieter check.

**Pre-commit enforcement.** Hooks block duplicate module-level definitions
(a real failure mode when a model appends rather than edits) and verify that
documented capabilities actually exist, because stale capability claims get
repeated back as fact.

## Failure modes I have actually hit

Listed because they are the specific risks of this method:

- **A patch that lands halfway and reports success.** I once reported an audit
  fix as shipped when only part of it had applied. Now: verify the file, not the
  tool's exit status.
- **Confident wrong causes.** See the Wi-Fi drop diagnosed as a thermal
  shutdown. Models are fluent about causes; fluency is not evidence.
- **A module-level function inserted into a class body.** It parsed cleanly and
  severed the class. Syntax validity is not correctness.
- **A scratch file shadowing a library.** A test script named after the library
  it imported silently took precedence, because the script's directory comes
  first on the path.
- **Undoing a deliberate decision.** A change "improved" a component by
  reintroducing a third-party service that had been deliberately retired for
  privacy reasons. The decision was in the documentation; the model had not read
  it. Architectural decisions must be recorded where they will be encountered.

## What this demonstrates

The useful skill here is not prompting. It is knowing what to distrust:
treating generated code as a submission from a fast, capable, and occasionally
overconfident contributor who has not been on-call for this system — and
building the harness that catches what review misses.
