# Subactor failure → Planfile / Koru development ticket bridge

**Status:** shipped (orchestrator escalator + `subactor-improvement` recorder)  
**Audience:** autonomy / Koru operators

## Why

Koru is the right **self-repair / development** loop (claim ticket → patch code →
TestQL / unit / quality gates). It is **not** a substitute for EQL / AQL / OQL /
URI, and it must **not** auto-mutate DNS, Plesk, secrets, or production apply
grants.

Subactor must therefore **emit** structured development tickets when execution
hits a *structural* failure, so Koru has something to pick up.

## Lifecycle

```text
Subactor task
  → structural error (e.g. invalid_runner_response, capability_unavailable,
    plan_hash_mismatch as code bug)
  → upsert development_defect (dedupe by fingerprint)
  → set blocked_by on the original ticket
  → Koru development queue
  → agent patches allowlisted code + regression tests
  → close development ticket
  → resume original Subactor task (preflight → AQL → dry-run → grant → Y/n)
```

Ops / HITL failures (`dns_mismatch`, `credential_missing`, `applied_unverified`,
apply-grant denials, etc.) stay **out** of the development queue.

## Hook payload

```json
{
  "type": "development_defect",
  "fingerprint": "orchestrator:stdout_json_truncated",
  "discovered_in": "PLF-364",
  "component": "orchestrator",
  "error_code": "invalid_runner_response",
  "affected_files": ["orchestrator/bin/subactor-run.mjs"],
  "acceptance_tests": [
    "output larger than 64 KiB remains valid JSON",
    "subactor ask reaches production Y/n prompt"
  ]
}
```

## Wiring

| Surface | Behaviour |
| --- | --- |
| Recipe `on_fail: ticket` | `runTask({ ticketEscalator })` — default CLI escalator upserts a real ticket; `stub: true` / missing hook → `stage: ticket_failed` (never `completed`) |
| `subactor-run` | Always installs `createDefaultTicketEscalator` |
| `platform/bin/subactor` | `record_improvement_failure` → `subactor-improvement record` |
| `subactor-improvement` | Planfile upsert on queue `development`; Koru autonomous loop |

Package: sibling checkout `~/github/subactor-improvement` (override with
`SUBACTOR_IMPROVEMENT_BIN`).

## Resume

After `subactor-improvement resolve SELFDEV-… --commit <sha>` marks the source
ticket ready:

```bash
subactor-run --ticket PLF-364
subactor-run --ticket PLF-364 --execute   # only after dry-run + grant + Y/n
```

Koru closing a development ticket does **not** bypass capability preflight,
AQL, dry-run, signed apply grant, or interactive confirmation.

## Related

- [`docs/koru.yaml`](../koru.yaml) — ticket iteration; no prod mutation commands
- [`autonomy-recommended-solution.md`](./autonomy-recommended-solution.md) — `on_fail: ticket`
- ADR-003 apply / HITL — grants remain mandatory on resume
