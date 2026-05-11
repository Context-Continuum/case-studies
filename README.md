# Case studies

Worked examples of substrate-boundary discipline applied to real
production bugs in a multi-agent cluster. Each case study walks the
same chain:

1. **Symptom** — what the operator (or a downstream system) observed
2. **Diagnosis** — the trace from symptom to first wrong line
3. **Substrate failure** — the deeper architectural pattern that
   allowed the bug to land
4. **Fix** — the change, placed at the right architectural boundary
5. **Regression test** — what guarantees the same bug class can't
   land again
6. **Receipt** — the durable record (inventory row, decision_id) so
   future agents can find the lesson

These aren't postmortems. They're the part of a postmortem worth
keeping — the substrate insight that survives the specific incident.

## Index

- **[F6 — wake-filter silently disabled cluster-wide](./F6-wake-filter-silent-disable.md)**
  A cross-cluster wake-routing module failed open on missing
  Firestore credentials. No log, no telemetry, no startup error —
  the cluster just went mute to cross-cluster pings for 12+ hours.
  Fix: auto-detect canonical creds path + refuse-at-init when still
  missing, emit a substrate signal the operator can see.

## What you'll see in these case studies (and what you won't)

You'll see: the diagnostic reasoning, the architectural judgment
about where the fix belongs, the doctrine application, and the
test discipline that locks the fix in place.

You won't see: the full implementation of the substrate primitives
that surfaced the bug. The trio observation substrate, the pull-
routing layer, the hook telemetry sink — those stay private. This
org's value is the operating discipline, not the codebase that
embodies it. The lock is here; the key isn't.

---

*See also: [operating-handbook](https://github.com/Context-Continuum/operating-handbook)
for the doctrines these case studies apply.*
