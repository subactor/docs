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
| Apply auth | Domain kill switch `PLESK_SYNC_APPLY`; dry-run planners | Kill switch + signed grant + immutable manifest | PR5a→5b→5c (see roadmap) | Founder token ≠ grant (ADR-003) |
| Transport | FTP path; SFTP needs paramiko in image | SFTP readiness gate | PR6 | FTP-only prod risk |
| DNS / verify | Desired state documented; Pages still serves docs | DNS→Plesk + fingerprint DoD | PR8–9 | **GitHub Pages is not healthy last_known_good** for content rollback |
| Vault | Lease via browser-agent vault in recipes | ADR-006 lease/revoke/audit | PR6–7 related | `.env` tokens in lab |

## PR0–PR4 evidence table

Commands run 2026-07-18 (platform workspace):

- `node scripts/check-component-drift.mjs` → **ok** (connectors SHA warn sibling≠submodule; content match)
- `node scripts/sync-intent-pack-derived.mjs --check` → **ok** (default mode is check; `--check` accepted)
- `node --test components/core/services/control/tests/intent-pack-registry.test.mjs` → **6/6 pass**
- `agents` `tests/nlp-uri-pack.test.mjs` → **4/4 pass**
- `orchestrator` `tests/pipeline.test.mjs` → **17/17 pass** (includes policy suite)

| Unit | Component commit (sibling) | Platform pin | Tests | Status |
| --- | --- | --- | --- | --- |
| **PR1** canonical paths + drift | ADR-007 docs; gate in platform | `platform` `fbd0692`; drift script | `check-component-drift.mjs` ok | **verified** |
| **PR2** intent pack registry | `core` `d44fbb2` (pack-first intents earlier in history); packs in `platform/config/intent-packs/` | core pin `d44fbb2`; agents `771053d` | intent-pack-registry 6/6 | **verified** (dual-run retained) |
| **PR3** phrase/LLM/step dedupe | `agents` `771053d`; sync script on platform | agents pin `771053d` | sync `--check` ok; nlp-uri-pack 4/4 | **partial** — pack SSOT for resolvers + derived artifacts; **Planfile imports still separate**; dual-run until PR10 |
| **PR4** recipe policy engine | `orchestrator` `d9b4599` | orchestrator not a compose submodule (CLI package) | pipeline 17/17 | **partial** — policy **core** done; `on_fail:rollback` = stub → must surface `rollback_failed`; ticket path needs real escalator (no stub success); compensation → PR7 |
| **PR5** grants / manifest | *draft only — uncommitted WIP* | not pinned | n/a for ship | **not shipped** — split **5a→5b→5c**; requires Accepted ADR-003 |

Honesty notes:

- **PR3 ≠ full migration.** Resolvers and derived YAML/JSON track packs; Planfile ticket YAML and some recipes remain hand-wired.
- **PR4 ≠ production-ready failure machine.** Retry/timeout/`on_fail` work; rollback does **not** execute compensation; ticket without hook must not yield `ok: true`.

## Fail-closed apply gates (target table; PR5)

| Gate | Env / artifact | Deny code (target) |
| --- | --- | --- |
| Master kill | `AUTONOMY_MUTATIONS_ENABLED=0` | `autonomy_mutations_disabled` |
| Domain kill | `PLESK_SYNC_APPLY≠1` (and master not `1`) | `plesk_sync_apply_required` |
| Grant | missing / bad sig / expired / wrong binding | `apply_grant_*` |
| Manifest | apply `plan_hash` ≠ dry-run | `plan_hash_mismatch` |

## PR5 draft reconciliation

Prior session drafted HMAC grants + grant-required apply in sibling trees (`runtime`/`connectors`/`core`/`urirun-connector-plesk`) **without** Accepted ADR-003. That work is **not** pushed as production behavior.

- **PR5a (next code):** immutable manifest + canonical JSON + `plan_hash`; apply must not free re-scan when hash bound.
- **PR5b:** signed grant per Accepted ADR-003.
- **PR5c:** replay / `jti` store.

Do **not** treat GitHub Pages as safe DNS/content rollback without noting it is an **unhealthy** last_known_good until Plesk cutover + verify (ADR-002/005).
