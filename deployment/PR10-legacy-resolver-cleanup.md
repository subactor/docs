# PR10 — legacy resolver / dual-run cleanup

**Status:** in progress (2026-07-18 continuation) — cold FALLBACK_PHRASES removed; dual-run stays **shadow** until metrics OK.  
**Why now:** PR9 production DNS cutover is blocked (G1 addon/docroot + G2/G6). Pack-first for docs/www is stable; SFTP readiness restored via urirun-node rebuild.

## Goal

Single pack SSOT end-to-end for docs/www intent resolution:

1. Pack registry resolves phrases (SSOT).
2. Docs/www adapters **fail closed** when packs dir is missing (no cold phrase list).
3. Dual-run compare remains default **shadow** (metrics / warn); not removed yet.

## `INTENT_PACK_DUAL_RUN`

| Value | Behaviour |
| --- | --- |
| `shadow` (**default**) | Pack-first; still compare legacy↔pack; attach `pack_compare.mode=shadow` (metrics / warn on mismatch). |
| `on` / `enforce` | Previous behaviour: always compare when registry loads. |
| `off` / `pack-only` | When pack hits: skip compare and (control) skip legacy path; `pack_compare.skipped=true`. |

Set on **hr-control** / agents runtimes, e.g. `INTENT_PACK_DUAL_RUN=shadow`.

## Done in this slice

- [x] Control bridge honors mode (`core` `intent-pack-bridge.mjs` + `routes/llm.mjs`).
- [x] Agents adapter honors mode (`agents` `intent-pack-adapter.mjs`).
- [x] Regression tests for `off` + default `shadow` / `on`.
- [x] **Remove cold FALLBACK_PHRASES** from docs/www adapters (fail closed if packs unmounted).
- [x] Step-catalog + planfile ticket process ids aligned with docs recipe (incl. `docs-publish-verify`).
- [x] Sync `--check` verifies planfile ticket uri_processes ids vs recipe.
- [x] Status docs updated honestly (PR9 still blocked; no DNS flip).

## Not done yet

- [ ] Delete YAML dual-run path entirely from agents (keep shadow dual-run until metrics OK).
- [ ] Full Planfile YAML auto-generated from packs (ids aligned; body still hand-maintained).
- [ ] Register `dns-record-reconcile` pack (still deferred — PR9 DNS connector).
- [ ] Flip default to `INTENT_PACK_DUAL_RUN=off` after shadow metrics stay clean.

## Safety

- Default `shadow` preserves observability without blocking pack-first.
- Use `INTENT_PACK_DUAL_RUN=on` temporarily if investigating a mismatch.
- Do **not** claim production docs publish success from this work.
- Packs must be mounted (`config/intent-packs` via `CONTROL_CONFIG_DIR`) in all control runtimes.
