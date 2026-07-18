# Autonomy implementation status (evidence)

**Date:** 2026-07-18  
**Nearest milestone:** Safe autonomous mutation in a **mocked** environment — not production `docs.subactor.com` deploy.  
**Roadmap:** [`../plans/autonomy-implementation-roadmap.md`](../plans/autonomy-implementation-roadmap.md)  
**ADRs:** [`adr/README.md`](./adr/README.md)

## CURRENT / TARGET / PLANNED / LEGACY

| Layer | CURRENT (implemented) | TARGET | PLANNED | LEGACY (still present) |
| --- | --- | --- | --- | --- |
| Intent | Pack registry + pack-first resolvers; derived phrases/LLM/step-catalog sync | Single pack SSOT end-to-end | Dual-run remove (PR10); Planfile auto from pack | Planfile imports hand-maintained; some dual-run compare |
| Policy | `on_fail` / retry / timeout / `depends_on` vs `after` in orchestrator | Full lifecycle + real compensation | `try_in_order`; release rollback (PR7) | Default `halt`; rollback = stub → `rollback_failed` |
| Apply auth | Kill switch `PLESK_SYNC_APPLY` + **immutable manifest / `plan_hash`** (PR5a) | Kill switch + signed grant + manifest | **PR5b** grant → **PR5c** jti | Founder token ≠ grant (ADR-003); grant-required **not** shipped |
| Transport | FTP path; SFTP needs paramiko in image | SFTP readiness gate | PR6 | FTP-only prod risk |
| DNS / verify | Desired state documented; Pages still serves docs | DNS→Plesk + fingerprint DoD | PR8–9 | **GitHub Pages is not healthy last_known_good** for content rollback |
| Vault | Lease via browser-agent vault in recipes | ADR-006 lease/revoke/audit | PR6–7 related | `.env` tokens in lab |

## PR0–PR5a evidence table

Commands run 2026-07-18:

- `urirun-connector-plesk` `pytest tests/test_plesk.py` → **28/28 pass**
- `testkit` `node --test tests/plesk-httpdocs-sync.test.mjs` → **8/8 pass**
- JS↔Python `plan_hash` parity on identical file list → **match**
- Smoke: `site_sync` dry-run on `www/` → `ok` + `plan_hash`; apply with wrong hash → `plan_hash_mismatch`, `files_uploaded=0`
- `node scripts/check-component-drift.mjs` → re-run after pin bump
- `node scripts/sync-intent-pack-derived.mjs --check` → **ok** (prior)
- intent-pack-registry / nlp-uri-pack / orchestrator pipeline → prior PR1–4 evidence still holds

| Unit | Component commit (sibling) | Platform pin | Tests | Status |
| --- | --- | --- | --- | --- |
| **PR1** canonical paths + drift | ADR-007 docs; gate in platform | `platform` `fbd0692`; drift script | `check-component-drift.mjs` ok | **verified** |
| **PR2** intent pack registry | `core` `d44fbb2` (pack-first intents earlier in history); packs in `platform/config/intent-packs/` | core pin `d44fbb2`; agents `771053d` | intent-pack-registry 6/6 | **verified** (dual-run retained) |
| **PR3** phrase/LLM/step dedupe | `agents` `771053d`; sync script on platform | agents pin `771053d` | sync `--check` ok; nlp-uri-pack 4/4 | **partial** — pack SSOT for resolvers + derived artifacts; **Planfile imports still separate**; dual-run until PR10 |
| **PR4** recipe policy engine | `orchestrator` `9dd8ed5` (policy core `d9b4599` + hardening) | orchestrator not a compose submodule (CLI package) | pipeline 20/20 | **partial→hardened** — `ticket_failed` / `rollback_failed`; retry clamp on mutate; compensation → PR7 |
| **PR5a** immutable manifest | `urirun-connector-plesk` + `connectors` bridge planner | connectors pin after ship | plesk pytest 28/28; testkit 8/8; smoke dry-run | **done** — apply requires matching `plan_hash` |
| **PR5b** signed apply grant | *not started* | — | — | **next** (ADR-003 crypto) |
| **PR5c** jti replay | *not started* | — | — | after 5b |

Honesty notes:

- **PR3 ≠ full migration.** Resolvers and derived YAML/JSON track packs; Planfile ticket YAML and some recipes remain hand-wired.
- **PR4 ≠ production-ready failure machine.** Retry/timeout/`on_fail` work; rollback does **not** execute compensation; ticket without hook must not yield `ok: true`.
- **PR5a ≠ grant auth.** Manifest binds dry-run→apply; kill switch still required; signed grants are **PR5b**.

## Fail-closed apply gates (CURRENT vs TARGET)

| Gate | Env / artifact | Deny code | Status |
| --- | --- | --- | --- |
| Domain kill | `PLESK_SYNC_APPLY≠1` | `plesk_sync_apply_required` | **CURRENT** |
| Manifest | apply `plan_hash` ≠ recomputed dry-run | `plan_hash_mismatch` | **CURRENT (PR5a)** |
| Master kill | `AUTONOMY_MUTATIONS_ENABLED=0` | `autonomy_mutations_disabled` | TARGET (PR5b wiring) |
| Grant | missing / bad sig / expired / wrong binding | `apply_grant_*` | TARGET (PR5b) |

## PR5 split

1. ADR-003 **Accepted** (crypto/TTL/replay/rotation/fail-closed).
2. Draft grant code discarded — do **not** ship grant-required apply in 5a.
3. **PR5a done** (this doc): immutable manifest + `plan_hash` in bridge + `urirun-connector-plesk`.
4. **Next: PR5b** signed apply grant verify → **PR5c** `jti` replay.

Do **not** treat GitHub Pages as safe DNS/content rollback without noting it is an
**unhealthy** last_known_good until Plesk cutover + verify (ADR-002/005).
