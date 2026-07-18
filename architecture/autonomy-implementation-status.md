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
| Apply auth | Dual kill switch + signed apply grant + `plan_hash` + **jti replay** (PR5a–5c) | production verify DoD | PR6 SFTP readiness | Founder token ≠ grant (ADR-003); dual-run retained |
| Transport | FTP path; SFTP needs paramiko in image | SFTP readiness gate | PR6 | FTP-only prod risk |
| DNS / verify | Desired state documented; Pages still serves docs | DNS→Plesk + fingerprint DoD | PR8–9 | **GitHub Pages is not healthy last_known_good** for content rollback |
| Vault | Lease via browser-agent vault in recipes | ADR-006 lease/revoke/audit | PR6–7 related | `.env` tokens in lab |

## PR0–PR5c evidence table

Commands run 2026-07-18 (PR5c):

- `@subactor/runtime` apply-grant + replay tests → **17/17 pass**
- `urirun-connector-plesk` `pytest tests/test_plesk.py` → **34/34 pass** (incl. jti replay: first OK / second deny / different jti OK)
- Deny: reused `jti` → `apply_grant_replay` with **zero** second upload
- Bridge planner still crypto-verifies without consuming (mutate path consumes)

| Unit | Component commit (sibling) | Platform pin | Tests | Status |
| --- | --- | --- | --- | --- |
| **PR1** canonical paths + drift | ADR-007 docs; gate in platform | `platform` `fbd0692`; drift script | `check-component-drift.mjs` ok | **verified** |
| **PR2** intent pack registry | `core` `d44fbb2` (pack-first intents earlier in history); packs in `platform/config/intent-packs/` | core pin `d44fbb2`; agents `771053d` | intent-pack-registry 6/6 | **verified** (dual-run retained) |
| **PR3** phrase/LLM/step dedupe | `agents` `771053d`; sync script on platform | agents pin `771053d` | sync `--check` ok; nlp-uri-pack 4/4 | **partial** — pack SSOT for resolvers + derived artifacts; **Planfile imports still separate**; dual-run until PR10 |
| **PR4** recipe policy engine | `orchestrator` `9dd8ed5` (policy core `d9b4599` + hardening) | orchestrator not a compose submodule (CLI package) | pipeline 20/20 | **partial→hardened** — `ticket_failed` / `rollback_failed`; retry clamp on mutate; compensation → PR7 |
| **PR5a** immutable manifest | `urirun-connector-plesk` `63a4fe1`; `connectors` `580ba39`; `testkit` `8675a5d` | connectors/testkit pins updated | plesk pytest; testkit; smoke dry-run | **done** |
| **PR5b** signed apply grant | runtime `31d2cb6`; core `011e763`; connectors `a42dcfc`; plesk `66be5c5`; testkit `531170c` | platform `385b24c` | runtime 12; plesk 33; testkit 9 | **done** — grant-required apply |
| **PR5c** jti replay | runtime `c6ba013`; plesk `cecfb36`; core `79d3178`; connectors `c578cc2` | platform `17740cc` | runtime 17; plesk 34 | **done** — single-use jti |

Honesty notes:

- **PR3 ≠ full migration.** Resolvers and derived YAML/JSON track packs; Planfile ticket YAML and some recipes remain hand-wired.
- **PR4 ≠ production-ready failure machine.** Retry/timeout/`on_fail` work; rollback does **not** execute compensation; ticket without hook must not yield `ok: true`.
- **PR5c ≠ DNS cutover.** Replay-safe grants in mock; SFTP/paramiko image readiness → **PR6**. No claim that `docs.subactor.com` is live on Plesk.

## Fail-closed apply gates (CURRENT)

| Gate | Env / artifact | Deny code | Status |
| --- | --- | --- | --- |
| Master kill | `AUTONOMY_MUTATIONS_ENABLED≠1` | `autonomy_mutations_disabled` | **CURRENT (PR5b)** |
| Domain kill | `PLESK_SYNC_APPLY≠1` | `plesk_sync_apply_required` | **CURRENT** |
| Grant | missing / bad sig / expired / wrong binding | `apply_grant_*` | **CURRENT (PR5b)** |
| Manifest | apply `plan_hash` ≠ recomputed dry-run | `plan_hash_mismatch` | **CURRENT (PR5a)** |
| Replay | reused `jti` | `apply_grant_replay` | **CURRENT (PR5c)** |

### Founder / admin grant path

After dry-run: `POST /api/apply-grants` (scope `plans:approve`) with `run_id`, `plan_hash`, `artifact_sha256`, `target`, pack, `risk_class`.  
HMAC: `APPLY_GRANT_HMAC_SECRET` (preferred) or `TOKEN_PEPPER`; rotation via `APPLY_GRANT_HMAC_SECRET_NEXT`.  
Pass returned `grant` as `apply_grant` on mutate. Each `jti` is single-use (replay store; optional `APPLY_GRANT_JTI_STORE` file). Secrets never in tickets/logs.

## PR5 split

1. ADR-003 **Accepted** (crypto/TTL/replay/rotation/fail-closed).
2. **PR5a done:** immutable manifest + `plan_hash`.
3. **PR5b done:** signed apply grant issue (control) + verify (bridge + plesk); `jti` issued.
4. **PR5c done:** `jti` replay store (consume on mutate; bridge verify-only).
5. **Next: PR6** SFTP/paramiko readiness (no DNS cutover yet).

Do **not** treat GitHub Pages as safe DNS/content rollback without noting it is an
**unhealthy** last_known_good until Plesk cutover + verify (ADR-002/005).
