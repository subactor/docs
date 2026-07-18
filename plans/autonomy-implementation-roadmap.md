# Roadmapa implementacji autonomii

**Status:** plan wdrożenia (Fazy 0–8 + kolejność PR).  
**Evidence CURRENT/TARGET:** [`../architecture/autonomy-implementation-status.md`](../architecture/autonomy-implementation-status.md)  
**Rekomendacja kanoniczna:** [`../architecture/autonomy-recommended-solution.md`](../architecture/autonomy-recommended-solution.md)  
**ADR:** [`../architecture/adr/README.md`](../architecture/adr/README.md) (**001–007 Accepted**)  
**Status ops:** [`../architecture/autonomy-ops-status-and-open-questions.md`](../architecture/autonomy-ops-status-and-open-questions.md)  
**Intent/fallbacki:** [`../architecture/intent-orchestration-and-fallbacks.md`](../architecture/intent-orchestration-and-fallbacks.md)  
**Publish docs:** [`docs-subactor-com-publish.md`](./docs-subactor-com-publish.md)

**Baseline diagnostyczny:** docs commit `5894906` — nie mieszać z refaktorem orchestratora.

**Nearest milestone:** safe autonomous mutation w **mocked** environment — nie production cutover DNS.

Dwa strumienie (równolegle, E2E dopiero po obu):

- **A** — platforma (packs, policy, grants, verify DoD)
- **B** — infrastruktura docs.subactor.com (DNS, TLS, SFTP, vault, releases)

---

## Faza 0 — dokumentacja i ADR

- Zachować `5894906` jako snapshot diagnostyczny.
- ADR-001–007 **Accepted** (grant crypto w ADR-003; rollback_failed / Pages caveat w ADR-005; lease/revoke w ADR-006).
- Evidence PR1–PR4: [`../architecture/autonomy-implementation-status.md`](../architecture/autonomy-implementation-status.md).
- Uwaga PR1: **rozstrzygnięte** — `platform/components/*` = submoduły (pin deploy);
  kanoniczny kod w `subactor/<name>`; drift: `platform/scripts/check-component-drift.mjs`
  ([`../architecture/canonical-component-paths.md`](../architecture/canonical-component-paths.md), ADR-007).

## Faza 1 — Intent Pack Registry

- Schemat packa w `platform/config/intent-packs/`.
- Pack **bez** `ALLOW`; tylko `required_capabilities` + CI ⊆ AQL.
- Wspólny loader (control, agents, LLM catalog, ticket generator, recipe validator).
- Migracja docs/www bez big-bang; dual-run legacy vs registry.
- **Unit 2–3 (verified / partial):** frazy docs/www SSOT w packach; control **pack-first**;
  `agents/nlp-uri-phrases.yaml` generowany; LLM fields sync z `situation_schema`;
  step-catalog annotated + `sync-intent-pack-derived.mjs --check`. Dual-run do PR10.
  Planfile imports **nie** przepisywane automatycznie (**PR3 = partial**).

## Faza 2 — Recipe Policy Engine

- Pola: `on_fail`, `depends_on` / `after`, `timeout_ms`, `retry`, `idempotency_key`, `compensation_step`.
- Domyślnie legacy `halt`; dopiero później `try_in_order`.
- **Unit 4 (partial / hardening):** policy core w `@subactor/orchestrator` — **done**.
  Retry tylko dla kroków idempotent/query (enforce). `on_fail:ticket` bez stub-success.
  `on_fail:rollback` → `rollback_failed` aż PR7. Compensation/`try_in_order` → PR7+.

## Faza 3 — Apply grants (split)

- **PR5a** — immutable manifest + canonical JSON + `plan_hash` (no free re-scan).
- **PR5b** — signed apply grant per ADR-003 (HMAC, issuer control, fail-closed).
- **PR5c** — `jti` replay store.
- Dual kill switch: `AUTONOMY_MUTATIONS_ENABLED` + `PLESK_SYNC_APPLY`.
- **Nie shipować grant-required** bez Accepted ADR-003 (już Accepted) i solidnego 5a.
- Draft kodu z wcześniejszej sesji = **WIP niepinowany** — patrz status evidence.

## Faza 4 — Connector / SFTP

- Paramiko w obrazie urirun-node; capability readiness; timeouty 15/120/180; błędy strukturalne.

## Faza 5 — Release deploy + rollback

- `releases/` + `current`/`previous`; `release-upload` / `activate` / `rollback`.

## Faza 6 — Vault

- Ownership wg ADR-006; brak credential → `needs_human`.

## Faza 7 — Migracja docs.subactor.com → Plesk

- Desired DNS w repo; vhost; SFTP; origin deploy z `__subactor_release.json`; cutover DNS; TLS; public verify; auto-rollback.
- Pages ≠ healthy content last_known_good (ADR-002/005).

## Faza 8 — Lifecycle stanów

- Pełna maszyna stanów planu; odpowiedź NL ze statusem bogatszym niż `ok`.

Szczegóły każdej fazy: dokument rekomendacji §3–§11.

---

## Kolejność PR {#kolejność-pr}

| PR | Zakres |
| -- | --- |
| 0 | Commit `5894906`, ADR-y i stan początkowy — **ADRs Accepted** |
| 1 | Kanoniczne ścieżki + drift gate — **verified** |
| 2 | Intent pack schema/registry — **verified** (dual-run) |
| 3 | Phrase/LLM/step dedupe onto packs — **partial** (Planfile still separate) |
| 4 | Recipe policy core — **partial** (rollback stub / ticket hardening) |
| 5a | Immutable manifest + plan_hash binding |
| 5b | Signed apply grants (ADR-003) |
| 5c | Grant replay (`jti`) |
| 6 | Paramiko/SFTP, capability readiness i strukturalne błędy |
| 7 | Release upload, activation i rollback |
| 8 | DNS/TLS preflight oraz public content fingerprint verify |
| 9 | Migracja `docs.subactor.com` z Pages do Pleska |
| 10 | Usunięcie legacy resolverów i starych kopii wiring |

Każdy PR odwracalny; kompatybilność ze starymi recipes do końca migracji.

**Uwaga projektowa:** w tym repo docs shipping = commit (+ push), bez otwierania GitHub PR — tabela „PR” oznacza **jednostki zmian implementacyjnych** w monorepo Subactor.

---

## Macierz testów (skrót)

Pełna lista: rekomendacja §12. Evidence: [`../architecture/autonomy-implementation-status.md`](../architecture/autonomy-implementation-status.md).

## Werdykt

Cztery fundamenty: intent pack SSOT · policy engine · connector (transport+rollback) · verify obowiązkowy.  
Reprezentatywny sukces produkcyjny: autonomiczny release docs na Plesk — **po** PR5–9.  
Najbliższy kamień: **safe mutate w mocku** (deny gates + manifest), nie cutover DNS.
