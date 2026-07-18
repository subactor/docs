# Roadmapa implementacji autonomii

**Status:** plan wdrożenia (Fazy 0–8 + kolejność PR).  
**Rekomendacja kanoniczna:** [`../architecture/autonomy-recommended-solution.md`](../architecture/autonomy-recommended-solution.md)  
**ADR:** [`../architecture/adr/README.md`](../architecture/adr/README.md)  
**Status ops:** [`../architecture/autonomy-ops-status-and-open-questions.md`](../architecture/autonomy-ops-status-and-open-questions.md)  
**Intent/fallbacki:** [`../architecture/intent-orchestration-and-fallbacks.md`](../architecture/intent-orchestration-and-fallbacks.md)  
**Publish docs:** [`docs-subactor-com-publish.md`](./docs-subactor-com-publish.md)

**Baseline diagnostyczny:** docs commit `5894906` — nie mieszać z refaktorem orchestratora.

Dwa strumienie (równolegle, E2E dopiero po obu):

- **A** — platforma (packs, policy, grants, verify DoD)
- **B** — infrastruktura docs.subactor.com (DNS, TLS, SFTP, vault, releases)

---

## Faza 0 — dokumentacja i ADR

- Zachować `5894906` jako snapshot diagnostyczny.
- ADR-y: zakres autonomii, DNS SSOT, HITL, DoD publish, rollback, sekrety.
- Wypchnąć docs; **bez** dużego refaktoru kodu w tym samym commitcie.
- Uwaga PR1: **rozstrzygnięte** — `platform/components/*` = submoduły (pin deploy);
  kanoniczny kod w `subactor/<name>`; drift: `platform/scripts/check-component-drift.mjs`
  ([`../architecture/canonical-component-paths.md`](../architecture/canonical-component-paths.md), ADR-007).

## Faza 1 — Intent Pack Registry

- Schemat packa w `platform/config/intent-packs/`.
- Pack **bez** `ALLOW`; tylko `required_capabilities` + CI ⊆ AQL.
- Wspólny loader (control, agents, LLM catalog, ticket generator, recipe validator).
- Migracja docs/www bez big-bang; dual-run legacy vs registry.
- **Unit 3 (w toku / done częściowo):** frazy docs/www SSOT w packach; control **pack-first**;
  `agents/nlp-uri-phrases.yaml` generowany; LLM fields sync z `situation_schema`;
  step-catalog `create_*_httpdocs_sync_ticket` annotated `recipe` + skeleton check
  (`platform/scripts/sync-intent-pack-derived.mjs`). Dual-run `pack_compare` zostaje do PR10.
  Planfile imports **nie** przepisywane automatycznie.

## Faza 2 — Recipe Policy Engine

- Pola: `on_fail`, `depends_on` / `after`, `timeout_ms`, `retry`, `idempotency_key`, `compensation_step`.
- Domyślnie legacy `halt`; dopiero później `try_in_order`.
- **Unit 4 (done):** `@subactor/orchestrator` `normalizeStep` + `runTask` honorują
  `on_fail` (`halt`|`continue`|`ticket`|`rollback` stub), `optional`/`required`,
  `timeout_ms`, `retry`, oraz rozróżnienie `depends_on` (sukces) vs `after` (zakończenie).
  Regresje w `orchestrator/tests/pipeline.test.mjs`. Live publish recipes bez zmian
  semantyki (fixtures w testach). Compensation/`try_in_order` → PR7 / później.

## Faza 3 — Apply grants

- `AUTONOMY_MUTATIONS_ENABLED` + signed grant (`plan_hash`, artefakt, target).
- Dry-run → immutable manifest; apply bez przeliczania plików.

## Faza 4 — Connector / SFTP

- Paramiko w obrazie urirun-node; capability readiness; timeouty 15/120/180; błędy strukturalne.

## Faza 5 — Release deploy + rollback

- `releases/` + `current`/`previous`; `release-upload` / `activate` / `rollback`.

## Faza 6 — Vault

- Ownership wg ADR-006; brak credential → `needs_human`.

## Faza 7 — Migracja docs.subactor.com → Plesk

- Desired DNS w repo; vhost; SFTP; origin deploy z `__subactor_release.json`; cutover DNS; TLS; public verify; auto-rollback.

## Faza 8 — Lifecycle stanów

- Pełna maszyna stanów planu; odpowiedź NL ze statusem bogatszym niż `ok`.

Szczegóły każdej fazy: dokument rekomendacji §3–§11.

---

## Kolejność PR {#kolejność-pr}

| PR | Zakres |
| -- | --- |
| 0 | Commit `5894906`, ADR-y i stan początkowy |
| 1 | Ustalenie kanonicznych ścieżek oraz kontrola kopii `platform/components` — **done** |
| 2 | Intent pack schema, registry i migracja docs/www — **done** (dual-run) |
| 3 | Deduplikacja phrase map, katalogu LLM i step-catalog — **done** (pack SSOT; sync script; dual-run until PR10) |
| 4 | `on_fail`, retry, timeout i statusy kroków w orchestratorze — **done** |
| 5 | Signed apply grants oraz plan/artifact hash binding |
| 6 | Paramiko/SFTP, capability readiness i strukturalne błędy |
| 7 | Release upload, activation i rollback |
| 8 | DNS/TLS preflight oraz public content fingerprint verify |
| 9 | Migracja `docs.subactor.com` z Pages do Pleska |
| 10 | Usunięcie legacy resolverów i starych kopii wiring |

Każdy PR odwracalny; kompatybilność ze starymi recipes do końca migracji.

**Uwaga projektowa:** w tym repo docs shipping = commit (+ push), bez otwierania GitHub PR — tabela „PR” oznacza **jednostki zmian implementacyjnych** w monorepo Subactor.

---

## Macierz testów (skrót)

Pełna lista: rekomendacja §12.

- Intent registry, policy engine, security (grants), connector, E2E NL→completed + scenariusze awarii.

## Werdykt

Cztery fundamenty: intent pack SSOT · policy engine · connector (transport+rollback) · verify obowiązkowy.  
Reprezentatywny sukces: autonomiczny release docs na Plesk (dry-run → grant → SFTP → activate → public fingerprint → rollback).
