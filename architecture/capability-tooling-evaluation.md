# Capability tooling evaluation (touri / uri2verify / TestQL / dockfra / hypervisor)

**Date:** 2026-07-18  
**Goal gap:** pack `required_capabilities` ↔ connector doctor readiness (Subactor / Plesk)  
**Scope:** smoke-test candidate packages; apply only what closes the gap; minimal refactors.

## Summary

| Package | Smoke result | Verdict for gap | Applied? |
| --- | --- | --- | --- |
| **tellmesh/touri** | Core import + `validate`/`list` OK; 14 pass / voice+markpact failures unrelated | **Helpful** — capability id + `.uri.capability.yaml` manifest shape | Yes (pattern + local manifests) |
| **tellmesh/uri2verify** | Unit tests pass; `capability-plan` was broken without hypervisor | **Helpful** — capability verification plan builder | Yes (touri fallback + used as design cue) |
| **semcod / TestQL** | Installed (`1.2.60`); `testql analyze` on platform OK | **Partial** — good scenario orchestrator, not SSOT for pack⊆doctor | Thin wrapper scenario only |
| **wronai/dockfra doctor** | CLI runs; reports wizard offline without dockfra stack | **Not useful** for Plesk capability ⊆ gate | No |
| **wronai/hypervisor** | Examples = touri capability registry copy; package not importable without install | **Partial** — contract-registry pattern; too heavy as dependency | No (uri2verify still prefers it when present) |
| **wronai/vdisplay** | Present; display orchestration | **Not useful** for this gap | No |

## Test results (detail)

### touri (`/home/tom/github/tellmesh/touri`)

- `pytest` (ignoring markpact): core registry/validate path works; voice optional extras fail without backends.
- CLI: `touri validate` / `touri list` on `examples/20_touri_capabilities` succeed.
- Takeaway: **capability ids + YAML manifests** are the right vocabulary for pack `required_capabilities`.

### uri2verify (`/home/tom/github/tellmesh/uri2verify`)

- Data-quality / replay / result-check tests: **pass**.
- Before refactor: `uri2verify capability-plan` → `ModuleNotFoundError: hypervisor.contract_registry`.
- After refactor: falls back to **touri** `*.uri.capability.yaml` registries; smoke OK on touri examples and on Subactor `platform/config/connector-capabilities/`.
- New regression: `tests/test_touri_capability_plan.py`.

### TestQL

- Installable from local oqlos tree; CLI works.
- Thin wrapper: `platform/components/testkit/tests/testql/capability-preflight.testql.toon.yaml` shells the preflight CLI (fixture green + red doctor → exit 1).
- Ownership (unchanged): TestQL may **call** the preflight CLI and assert exit codes; it must not own capability SSOT ([testing-intents-and-deploy-results.md](./testing-intents-and-deploy-results.md)).

### dockfra doctor

- `dockfra cli doctor` → wizard offline (expected without running dockfra root).
- Diagnoses Docker Compose wizard health, not connector capability readiness → **out of scope**.

### hypervisor

- `examples/20_touri_capabilities` mirrors touri manifests (same pattern).
- `pytest` collection fails without editable install (`No module named 'hypervisor'`).
- Useful as optional backend for uri2verify when `contracts/` exists; not required for Subactor preflight.

## Integration shipped in Subactor

| Artefact | Role |
| --- | --- |
| `platform/config/connector-capabilities/catalog.v1.json` | Pack id ↔ **live** doctor keys (`sftp`, `tls_san_check`, `ssl_ensure`, …) |
| `platform/config/connector-capabilities/plesk.doctor.fixture.json` | CI fixture mirroring live short-key + `available` shape |
| `platform/config/connector-capabilities/*.uri.capability.yaml` | Touri-style docs (sync / sftp / tls / ssl_ensure) |
| `platform/config/connector-capabilities/preflight.mjs` | ⊆ library + live fetch (urirun `/run`, bridge `/processes/run`) |
| `platform/scripts/capability-preflight.mjs` | CLI (`--live`, `--via bridge\|urirun`, `capability_unavailable`) |
| `platform/test/capability-preflight.test.mjs` | Unit/regression |
| `core/.../capability-preflight-gate.mjs` | Fail-closed gate for control |
| `core/.../routes/llm.mjs` + `plans.mjs` | Deny before NL success / propose-from-intent |
| `platform/bin/subactor` | Surfaces `capability_unavailable` / `preflight_failed` on `ask` |

```bash
# Fixture (CI)
node platform/scripts/capability-preflight.mjs --json
node platform/scripts/capability-preflight.mjs --require-publish-ready

# Live doctor from running urirun-node
URIRUN_NODE_TOKEN=$SUBACTOR_ADMIN_TOKEN \
  node platform/scripts/capability-preflight.mjs --live --via urirun --urirun-url http://127.0.0.1:18765 --json

# Live via bridge (in-compose)
BRIDGE_INTERNAL_URL=http://hr-bridge:8081 BRIDGE_SERVICE_TOKEN=… \
  node platform/scripts/capability-preflight.mjs --live --via bridge --json
```

**Live doctor URI:** `plesk://host/doctor/query/report`  
Shape: short keys + `{available|boolean}` + `production_publish_ready`. Pack ids use catalog aliases; `plesk.site.sync` is derived when SFTP is ready.

**Packs (docs/www)** declare: `plesk.site.sync`, `plesk.transport.sftp`, `plesk.tls_san_check`, `plesk.ssl_ensure`.  
`letsencrypt` stays **not** required — never claim public LE success.

**Control env:** `CAPABILITY_PREFLIGHT=1` (default), `CAPABILITY_PREFLIGHT_LIVE=1` (bridge/urirun when configured; else fixture). Inject `CAPABILITY_DOCTOR_PATH` or test `doctorReport` for red-path smoke.

## What was refactored

1. **uri2verify** — `capability-plan` CLI: hypervisor when `contracts/` present, else **touri registry**.
2. **platform** — live doctor normalize (`available`↔`ready`), CLI `--live`, pack/catalog SSL+SFTP alignment.
3. **core/control** — fail-closed on `/api/llm/intent` + `/api/plans/propose-from-intent`.

No dockfra / vdisplay / hypervisor wholesale changes. **No production DNS flip / no LE public success claim.**

## Test table (this continuation)

| Check | Result |
| --- | --- |
| Unit fixture ⊆ packs | PASS |
| Unit live-shape normalize (`available` + short keys) | PASS |
| Unit red sftp → `capability_unavailable` | PASS |
| CLI fixture exit 0 / red exit 1 | PASS |
| Gate mocked red → deny (`model_name: null`) | PASS |
| Live `--via urirun` against stack | PASS (`doctor_source: urirun-live`, both packs ok) |
| TestQL thin wrapper file | Added (shells CLI) |

## Remaining gaps

1. **AQL ⊆ check** — packs declare capabilities; still need CI that every required id is allowed by AQL contracts (separate from doctor readiness).
2. **Apply-grant path** — intent/propose gated; optional extra deny on `POST /api/apply-grants` for defense in depth.
3. **touri voice/markpact flakes** — unrelated; leave for tellmesh maintainers.

## Polish one-liner

Live `plesk://host/doctor/query/report` feeds capability-preflight; control/`subactor ask` fail-closed on red caps (`capability_unavailable` / `preflight_failed`) — no success promise / no apply expand when SFTP/SSL doctor keys are red.
