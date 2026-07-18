# PR10 — legacy resolver / dual-run cleanup

**Status:** started (2026-07-18) — safe reduction only; **not** full removal of cold fallbacks.  
**Why now:** PR9 production DNS cutover is blocked (G1/G2/G6 red). Pack-first for docs/www is already stable; dual-run can move to shadow / opt-out.

## Goal

Single pack SSOT end-to-end for docs/www intent resolution:

1. Pack registry resolves phrases (SSOT).
2. Legacy adapters (`docs-sync-intent` / `www-sync-intent` / YAML phrase map) remain as **cold fallback** when packs dir is missing.
3. Dual-run compare is no longer mandatory on every request.

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
- [x] Status docs updated honestly (PR9 still blocked; no DNS flip).

## Not done yet

- [ ] Remove cold FALLBACK_PHRASES from docs/www adapters (keep until packs always mounted in all envs).
- [ ] Planfile ticket YAML auto-generated from packs.
- [ ] Delete YAML dual-run path entirely from agents.
- [ ] Register `dns-record-reconcile` pack (still deferred — PR9 DNS connector).

## Safety

- Default `shadow` preserves observability without blocking pack-first.
- Use `INTENT_PACK_DUAL_RUN=on` temporarily if investigating a mismatch.
- Do **not** claim production docs publish success from this work.
