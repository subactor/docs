# Capability tooling evaluation (touri / uri2verify / TestQL / dockfra / hypervisor)

**Date:** 2026-07-18  
**Goal gap:** pack `required_capabilities` ↔ connector doctor readiness (Subactor / Plesk)  
**Scope:** smoke-test candidate packages; apply only what closes the gap; minimal refactors.

## Summary

| Package | Smoke result | Verdict for gap | Applied? |
| --- | --- | --- | --- |
| **tellmesh/touri** | Core import + `validate`/`list` OK; 14 pass / voice+markpact failures unrelated | **Helpful** — capability id + `.uri.capability.yaml` manifest shape | Yes (pattern + local manifests) |
| **tellmesh/uri2verify** | Unit tests pass; `capability-plan` was broken without hypervisor | **Helpful** — capability verification plan builder | Yes (touri fallback + used as design cue) |
| **semcod / TestQL** | Installed (`1.2.60`); `testql analyze` on platform OK | **Partial** — good scenario orchestrator, not SSOT for pack⊆doctor | No (documented only; wrap CLI later) |
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
- `testql analyze` discovers platform TestQL scenarios.
- Ownership (unchanged): TestQL may **call** the preflight CLI and assert exit codes; it must not own capability SSOT ([testing-intents-and-deploy-results.md](./testing-intents-and-deploy-results.md)).

### dockfra doctor

- `dockfra cli doctor` → wizard offline (expected without running dockfra root).
- Diagnoses Docker Compose wizard health, not connector capability readiness → **out of scope**.

### hypervisor

- `examples/20_touri_capabilities` mirrors touri manifests (same pattern).
- `pytest` collection fails without editable install (`No module named 'hypervisor'`).
- Useful as optional backend for uri2verify when `contracts/` exists; not required for Subactor preflight.

## Integration shipped in Subactor

Minimal local gate (no foreign monorepo rewrite):

| Artefact | Role |
| --- | --- |
| `platform/config/connector-capabilities/catalog.v1.json` | Pack capability id ↔ doctor keys / aliases |
| `platform/config/connector-capabilities/plesk.doctor.fixture.json` | CI doctor readiness fixture (`production_publish_ready` + caps) |
| `platform/config/connector-capabilities/*.uri.capability.yaml` | Touri-style docs for `plesk.site.sync` / `plesk.transport.sftp` |
| `platform/config/connector-capabilities/preflight.mjs` | ⊆ check library |
| `platform/scripts/capability-preflight.mjs` | CLI gate (`capability_unavailable` on miss) |
| `platform/test/capability-preflight.test.mjs` | Regression (green fixture, red sftp, alias map, CLI exit codes) |

```bash
node platform/scripts/capability-preflight.mjs --json
node platform/scripts/capability-preflight.mjs --require-publish-ready
node platform/scripts/capability-preflight.mjs --doctor /path/to/live-doctor.json
```

Live doctor JSON should use the same shape as the fixture (`capabilities.<id>.ready`). Short keys (`sftp`) resolve via catalog aliases.

## What was refactored

1. **uri2verify** — `capability-plan` CLI: hypervisor when `contracts/` present, else **touri registry**; adapters `touri_manifests_to_capability_records` / `build_capability_test_plan_from_touri`; hypervisor contract test skipped when unavailable.
2. **platform** — Makefile / `test:meta` include capability-preflight tests; new connector-capabilities config + script.

No dockfra / vdisplay / hypervisor wholesale changes.

## Remaining gaps

1. **Live doctor producer** — fixture is CI SSOT; wire urirun-node / bridge to emit the same JSON (paramiko, sync OQL, verify probes) and pass `--doctor` in ops preflight.
2. **Orchestrator hook** — call preflight before NL success promise / grant (deny `capability_unavailable`).
3. **AQL ⊆ check** — packs declare capabilities; still need CI that every required id is allowed by AQL contracts (separate from doctor readiness).
4. **TestQL thin wrapper** — optional scenario that shells the preflight CLI (not blocking).
5. **touri voice/markpact flakes** — unrelated to this gap; leave for tellmesh maintainers.

## Polish one-liner

Przetestowano touri, uri2verify, TestQL, dockfra doctor i wzorce hypervisor: **pomocne są touri (manifesty) i uri2verify (plan weryfikacji)**; TestQL częściowo; dockfra/vdisplay nie. Domknięto lukę lokalnym gate’em pack⊆doctor w platformie + małym fallbackiem touri w uri2verify.
