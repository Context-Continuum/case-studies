# F6 — wake-filter silently disabled cluster-wide

> Substrate-boundary discipline applied at a Monitor-task init
> boundary. Live receipt — closed 2026-05-10 in [PR #91 / commit
> `8cbbd58`](https://github.com/ExtractOS/mission-control/pull/91)
> over a 3h 44m window that also closed 8 other findings.

## Context

The cluster runs a *wake filter* on each agent's machine — a long-
running Python process that polls two surfaces:

1. The **local scratchpad daemon** (for in-cluster messages), and
2. **Firestore items with `_xClusterTo` field** (for cross-cluster
   pings from the peer cluster's agents)

It emits one stdout line per message that should wake *this* agent.
Claude Code's Monitor tool watches that stdout and surfaces each
line as a task-notification. It's the load-bearing routing layer
between async cluster traffic and synchronous agent attention.

Firestore polling is opt-in via two env vars:
`PHASESHIFT_WAKE_FIRESTORE_CREDS_PATH` (path to the SA JSON) and
`PHASESHIFT_WAKE_CLUSTER_ID` (which cluster we are). Set them, get
cross-cluster wake. Unset them, daemon-only operation — the design
intent for installs that don't need cross-cluster wake yet.

That's the design. Then this happened.

## The symptom

Operator ran a phone-share rickroll test — shared a video URL from
the Mission Control PWA addressed to a specific agent on the peer
side. The cross-cluster bridge stamped the item with
`_xClusterTo: ["Mac/Claude-A"]`. Mac/A's wake filter should have
picked it up and woken him. Mac/A never received it.

Wake filter on Mac/A: `tasklist` showed the python process running.
Cursor file fresh. Daemon polling stream looked healthy. Stderr
silent except for the daemon-side poll lines. *Looked* armed.
Actually mute.

The same was true on the Windows side. **Cluster-wide silent
failure on cross-cluster pings** — Mac/A diagnosed his own side
silent, Win confirmed independently. No alert, no error, no
indicator anywhere that cross-cluster wake had degraded.

## Diagnosis

The wake filter's old startup path (pre-fix):

```python
def init_firestore_client():
    creds_path = os.environ.get("PHASESHIFT_WAKE_FIRESTORE_CREDS_PATH")
    cluster_id = os.environ.get("PHASESHIFT_WAKE_CLUSTER_ID")
    if not creds_path or not cluster_id:
        return None  # silent
    try:
        import firebase_admin
        ...
    except ImportError:
        return None  # silent
```

Returning `None` made every Firestore-polling call site a no-op:

```python
def poll_firestore(self):
    if self.firestore is None:
        return
    ...
```

The root cause wasn't "operator forgot to set the env var." It was
**the Monitor primitive doesn't inherit shell-exported env vars
from `run-bridge.sh`**. When the operator armed the wake filter via
`Monitor("python tools/cluster_wake_filter.py")` from a fresh post-
reboot Claude Code session, that subprocess started with an empty-
ish env (Claude Code's spawning context, not the operator's
interactive shell). `run-bridge.sh`'s `export
PHASESHIFT_WAKE_FIRESTORE_CREDS_PATH=...` never reached the
subprocess.

So the wake filter started, hit the early-return branch, set
`self.firestore = None`, and ran forever as daemon-only. Cursor
kept updating (daemon side worked). Wake events that needed
Firestore visibility simply never fired.

Same failure on every machine that armed the wake filter via
Monitor without first manually re-exporting the env vars inside the
Monitor command itself. Cluster-wide silent failure with no
observable signal.

## The substrate failure underneath the bug

The bug isn't the env var. It's the **wake filter's posture toward
missing input**.

A wake filter is a routing layer. Its job is to deliver direct-
addressed messages to the addressee. If it cannot do that job —
for any reason — every downstream consumer of routing needs to
know. Silent degradation isn't acceptable in routing layers; it
produces exactly the symptom above (everything looks fine, nobody
knows it's broken).

Two architectural patterns were missing:

1. **Auto-detect canonical paths.** The cluster has an install
   convention: bridge SA credentials live at
   `~/.extractos-bridge/credentials.json`. The wake filter reuses
   those credentials for Firestore polling. Asking the operator to
   also set `PHASESHIFT_WAKE_FIRESTORE_CREDS_PATH` *additionally*
   was duplicate configuration — a step guaranteed to be forgotten,
   and worse, a step that **already worked when the operator was
   running interactively** but broke when the Monitor primitive
   spawned the same script from a different parent process.

2. **Refuse-at-init, not refuse-at-runtime.** If after auto-detect
   the creds are *still* genuinely missing, the wake filter must
   emit a startup-time substrate signal — a stderr WARNING the
   operator sees, and a structured `hook_telemetry` record any
   future audit can query. The filter can still proceed as daemon-
   only (intentional for some installs), but the operator gets to
   make that choice *knowingly* instead of inheriting it from a
   silent fall-through.

These are two instances of the doctrine we call **SUBSTRATE-
BOUNDARY DISCIPLINE**: enforce invariants at the init boundary
rather than asking the operator to remember and verify at runtime.
F6 was the 12th application of this doctrine in a single ~24h
session of substrate work.

## The fix

Real code from `tools/cluster_wake_filter.py:485-614`. Two layered
changes inside the wake filter's startup path.

**Layer 1 — auto-detect from canonical bridge install path
(lines 493-555):**

```python
DEFAULT_FIRESTORE_CREDS_PATH = (
    pathlib.Path.home() / ".extractos-bridge" / "credentials.json"
)

def _resolve_firestore_creds_path() -> pathlib.Path | None:
    """Order of preference:
      1. PHASESHIFT_WAKE_FIRESTORE_CREDS_PATH env (explicit override)
      2. ~/.extractos-bridge/credentials.json (bridge's canonical install)
      3. None — disabled, with warning + telemetry
    """
    override = os.environ.get("PHASESHIFT_WAKE_FIRESTORE_CREDS_PATH")
    if override:
        override_path = pathlib.Path(override)
        if override_path.is_file():
            return override_path
        # Override set but file missing — fall through to auto-detect.
        print(
            f"[wake-filter] PHASESHIFT_WAKE_FIRESTORE_CREDS_PATH={override} "
            f"does not exist; falling back to auto-detect at "
            f"{DEFAULT_FIRESTORE_CREDS_PATH}.",
            file=sys.stderr,
        )

    if DEFAULT_FIRESTORE_CREDS_PATH.is_file():
        if not override:
            print(
                f"[wake-filter] PHASESHIFT_WAKE_FIRESTORE_CREDS_PATH unset; "
                f"auto-detected creds at {DEFAULT_FIRESTORE_CREDS_PATH} "
                f"(set env var to override).",
                file=sys.stderr,
            )
        return DEFAULT_FIRESTORE_CREDS_PATH

    # Layer 2 — refuse-at-init substrate signal
    print(
        "[wake-filter] WARNING: Firestore wake DISABLED — no creds at "
        f"$PHASESHIFT_WAKE_FIRESTORE_CREDS_PATH or "
        f"{DEFAULT_FIRESTORE_CREDS_PATH}. "
        "Cross-cluster wakes (Mission Control cluster_messages, "
        "foreign-cluster items) will not surface. ...",
        file=sys.stderr,
    )
    _emit_firestore_init_failure_telemetry(
        reason="no_creds_found",
        details=(
            f"override={os.environ.get('PHASESHIFT_WAKE_FIRESTORE_CREDS_PATH', '')!r} "
            f"autodetect_path={str(DEFAULT_FIRESTORE_CREDS_PATH)!r}"
        ),
    )
    return None
```

**The B1 telemetry record** (`_emit_firestore_init_failure_telemetry`,
lines 558-583) lands a structured row in our hook-telemetry sink:

```python
{
    "hook_name": "cluster_wake_filter",
    "event": "error",
    "source_trigger": "exception",
    "session_id": "monitor-process",   # wake filter isn't a Claude session
    "ts_ms": now_ms(),
    "error_repr": f"firestore_init_failed: reason={reason} {details}",
}
```

Two effects:

- **Most machines never see the warning.** Auto-detect finds
  `~/.extractos-bridge/credentials.json` and the filter starts
  fully armed. The Monitor-doesn't-inherit-env issue stops
  mattering because env wasn't load-bearing anymore.

- **The remaining machines** that genuinely don't have creds see a
  loud stderr message AND emit a structured event the next
  cluster-health audit will surface. Post-merge, operators can run:

  ```bash
  jq 'select(.hook_name=="cluster_wake_filter" and .event=="error")' \
      ~/.claude/state/hook_telemetry.jsonl
  ```

  ... to see exactly when Firestore polling has degraded —
  *without waiting for an operator-caught test to surface it*.

## Regression tests

Seven acceptance tests at `tests/acceptance/wake_filter/test_firestore_creds_autodetect.py`, all passing in CI:

| Test | Pins |
|---|---|
| `test_autodetect_succeeds_when_env_unset_and_default_exists` | Auto-detect happy path |
| `test_autodetect_disabled_when_env_unset_and_default_missing` | Refuse-at-init AND B1 telemetry emit |
| `test_env_override_takes_precedence_when_file_exists` | Explicit override beats auto-detect |
| `test_env_override_falls_back_to_autodetect_when_override_missing` | Defensive fall-through |
| `test_env_override_missing_and_no_autodetect_returns_disabled` | Disabled state honest, not silent |
| `test_default_path_points_at_bridge_install` | Shape-pin against silent divergence from bridge's canonical SA path |
| `test_empty_string_env_falls_through_to_autodetect` | Bash convention: empty string = unset |

Full mission-control suite at the merge of PR #91: **1213 passed /
2 skipped / 0 failures.** Wake-filter acceptance suite specifically:
30/30 passed.

## Receipt

| Artifact | Value |
|---|---|
| PR | [#91](https://github.com/ExtractOS/mission-control/pull/91) |
| Merge commit | `8cbbd5821f8369cb4868f063613cd2026431dbe8` |
| Merged | 2026-05-11T06:52:07Z |
| Lines of code changed | +327 / -11 |
| `decision_id` | `cluster_substrate_boundary_discipline_v0` |
| Substrate-boundary instance # | 12th in single 3h 44m session |
| Inventory row | `mission-control/docs/SOP_AUTOMATION_INVENTORY.md` |
| Doctrine cross-refs | `cluster_wake_filter_must_use_monitor_primitive`, `cluster_wake_reliability_harness_v0` |
| Lane closure status | F1-F6 of the wake-harness lane all closed (table in PR body) |

## What this case study demonstrates

> When a routing layer goes mute, ask "what does it surface when it
> can't do its job?" If the answer is "nothing," the fix isn't in
> the polling logic. The fix is at the init boundary — make the
> layer refuse to start cleanly without telling the operator what's
> missing, and emit a queryable substrate signal so the next
> cluster-health audit can find every silent-degradation case
> without waiting for someone's phone-share test to surface it.

The whole reason this bug ran undiagnosed is that the wake filter
looked healthy from every observable surface. The fix restores
**observability where it was missing**, not just behavior.

Silent-degrade-on-missing-config is the most pernicious bug class
in agent operations because nothing tells you it's happening.
Refuse-at-init + auto-detect canonical paths + a structured
telemetry record together close the failure mode at the init
boundary — and make every future silent-degradation case land in a
`jq`-able audit trail instead of a cluster-wide phantom outage.

---

*This case study is one of nine substrate findings closed in a
single 3h 44m session (2026-05-10 21:38 → 2026-05-11 01:23). Each
landed at the right architectural boundary the first time, which
is why the pace was possible. See the [profile](https://github.com/Context-Continuum)
for the full F1–F10 chain.*
