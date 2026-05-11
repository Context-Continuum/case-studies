# F6 — wake-filter silently disabled cluster-wide

## Context

The cluster runs a *wake filter* on each agent's machine — a long-
running Python process that polls two surfaces (a local scratchpad
daemon and a Firestore-backed cross-cluster channel) and emits a
notification when a post is addressed to that agent. It's the
load-bearing routing layer that lets one agent ping another by
name without spam-waking the whole cluster on every routine post.

Firestore polling is opt-in via two env vars. When set, the filter
authenticates with a Workspace service-account JSON and watches
`items/` documents that have a `_xClusterTo` field naming this
cluster. When unset, the filter is "daemon-only" — backwards-compat
for installs that don't need cross-cluster wake.

That's the design. Then this happened.

## Symptom

Operator-flagged, paraphrased from the scratchpad:

> "I sent a direct-addressed cross-cluster ping to your agent ~12
> hours ago. He's been silent the whole time. Is the wake filter
> running?"

Initial agent self-report: "Wake filter `bp62vfvca` task is
running per `tasklist`, healthy, polling every 5 seconds." Cursor
file fresh. Daemon side fine.

Hours of cluster-wide muteness with no surfaced error anywhere.

## Diagnosis

The wake filter's startup path:

```python
def __init__(self, ...):
    try:
        creds_path = os.environ.get("PHASESHIFT_WAKE_FIRESTORE_CREDS_PATH")
        cluster_id = os.environ.get("PHASESHIFT_WAKE_CLUSTER_ID")
        if creds_path and cluster_id:
            self.firestore = init_firestore(creds_path)
            self.cluster_id = cluster_id
        else:
            self.firestore = None  # ← the bug
    except Exception as e:
        log.warning(f"Firestore init failed: {e}")
        self.firestore = None
```

If either env var is unset, `self.firestore` becomes `None` and
the Firestore polling code paths are no-ops:

```python
def poll_firestore(self):
    if not self.firestore:
        return  # ← silent skip
    ...
```

Operator's env had `PHASESHIFT_WAKE_FIRESTORE_CREDS_PATH` unset
on this machine. The wake filter started, fell into the
`firestore = None` branch, and ran forever as daemon-only without
ever surfacing a signal that it had degraded.

The cursor file kept updating (daemon polling worked fine), so
"is the wake filter alive?" looked healthy. Wake events that
needed Firestore visibility simply never fired.

## The substrate failure underneath the bug

The bug isn't the missing env var. The bug is the wake filter's
posture toward missing input.

A wake filter is a *routing layer*. Its job is to make sure
direct-addressed messages reach the addressee. If it cannot do
that job — for any reason — every downstream consumer of routing
needs to know. Silent degradation isn't acceptable in routing
layers; it's a class of failure that produces exactly the symptom
above ("everything looks fine, nobody knows it's broken").

Two architectural patterns were missing:

1. **Auto-detect canonical paths.** The cluster has an
   *install convention*: bridge credentials live at
   `~/.extractos-bridge/credentials.json`. The wake filter reuses
   those creds for Firestore polling. Asking the operator to also
   set `PHASESHIFT_WAKE_FIRESTORE_CREDS_PATH` to that same path
   duplicates configuration — a step that *will* get forgotten on
   one machine eventually. The wake filter should check the
   canonical path *before* concluding creds are missing.

2. **Refuse-at-init, not refuse-at-runtime.** If after auto-detect
   the creds are still genuinely missing, the wake filter shouldn't
   silently disable. It should emit a startup-time substrate signal
   — a stderr WARNING the operator sees on launch, and a structured
   `hook_telemetry` record any future audit can query. The filter
   *can* still proceed as daemon-only (some installs intentionally
   skip Firestore), but the operator gets to make that choice
   knowingly instead of inheriting it from a missing env var.

The doctrine being applied is what we call **SUBSTRATE-BOUNDARY
DISCIPLINE** in the operating handbook: enforce invariants at the
write boundary (or in this case, the init boundary) rather than
asking the operator to remember and check at runtime.

## Fix

Two layered changes inside the wake filter's init path:

```python
def __init__(self, ...):
    creds_path = os.environ.get("PHASESHIFT_WAKE_FIRESTORE_CREDS_PATH")
    cluster_id = os.environ.get("PHASESHIFT_WAKE_CLUSTER_ID")

    # Layer 1 — auto-detect from canonical bridge install path
    if not creds_path:
        canonical = Path.home() / ".extractos-bridge" / "credentials.json"
        if canonical.exists():
            creds_path = str(canonical)
            log.info(f"auto-detected Firestore creds at {canonical}")

    if creds_path and cluster_id:
        self.firestore = init_firestore(creds_path)
        self.cluster_id = cluster_id
    else:
        # Layer 2 — refuse-at-init substrate signal
        self.firestore = None
        sys.stderr.write(
            "WARNING: wake filter starting without Firestore polling. "
            "Cross-cluster pings will NOT wake this agent. "
            "Set PHASESHIFT_WAKE_FIRESTORE_CREDS_PATH + "
            "PHASESHIFT_WAKE_CLUSTER_ID to enable, or install creds at "
            f"{Path.home() / '.extractos-bridge' / 'credentials.json'}.\n"
        )
        hook_telemetry.emit(
            event="wake_filter_degraded",
            reason="firestore_creds_missing",
            severity="warning",
        )
```

Two effects:

- Most machines never see the warning — auto-detect finds the
  canonical path and the filter starts fully armed.
- The remaining machines that genuinely don't have creds see a
  loud stderr message AND emit a structured event the next
  cluster-health audit will surface.

## Regression test

Seven tests at the wake-filter test boundary:

1. `test_autodetect_finds_canonical_path` — given creds at the
   canonical path + env var unset, filter initializes with
   `self.firestore != None`
2. `test_env_var_overrides_canonical` — given creds at both a
   custom path (env var) and canonical (file), env var wins
3. `test_no_creds_emits_stderr_warning` — given no creds at
   either location, stderr contains "wake filter starting without
   Firestore polling"
4. `test_no_creds_emits_hook_telemetry` — same scenario, a
   telemetry record with `event=wake_filter_degraded` lands in
   the sink
5. `test_no_creds_proceeds_daemon_only` — same scenario, the
   filter still starts (just with `self.firestore = None`)
6. `test_corrupted_canonical_falls_through` — canonical file
   exists but is malformed JSON; filter emits the same WARNING +
   telemetry, proceeds daemon-only
7. `test_unreadable_canonical_falls_through` — canonical file
   exists but isn't readable (permissions); same fallthrough

The tests pin the *convention* (auto-detect-canonical, refuse-at-
init-with-substrate-signal) at the public init surface, so a
future change to internal mechanics can't accidentally
re-introduce silent degradation.

## Receipt

- A row was added to the operating cluster's SOP automation
  inventory tagging this as an instance of SUBSTRATE-BOUNDARY
  DISCIPLINE applied at an init boundary
- `decision_id: cluster_substrate_boundary_discipline_v0` updated
  to reference this finding (we number them F1, F2, ... — this
  one's F6)
- A second F-finding (F10) in the same family of "wrapper scripts
  silently degrade when canonical config isn't found" was closed
  later the same day using the same auto-detect-canonical-path
  pattern, validating the doctrine's generalizability

## What this case study demonstrates

The substrate-not-proxy principle and the substrate-boundary
discipline together produce a specific debugging move:

> When a routing layer goes mute, ask "what does it surface when
> it can't do its job?" If the answer is "nothing," the fix isn't
> in the polling logic. The fix is at the init boundary — make
> the layer refuse to start cleanly without telling the operator
> what's missing.

The whole reason this bug ran for 12 hours is that the wake
filter looked healthy from every observable surface. The fix
restores observability where it was missing, not just behavior.

Silent-degrade-on-missing-config is the most pernicious bug
class in distributed systems because nothing tells you it's
happening. Refuse-at-init + auto-detect canonical paths together
close the failure mode at the architectural boundary instead of
asking every operator to remember every env var.
