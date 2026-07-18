# docs.subactor.com

Publiczna dokumentacja Subactor.

## Autonomy (CLI → connectors)

| Dokument | Opis |
|----------|------|
| [architecture/autonomy-recommended-solution.md](architecture/autonomy-recommended-solution.md) | **Rekomendacja kanoniczna:** kontrolowany katalog zdolności, strumienie A/B, fazy, werdykt 4 fundamentów |
| [architecture/adr/README.md](architecture/adr/README.md) | ADR Phase 0: zakres autonomii, DNS SSOT, HITL, DoD publish, rollback, sekrety |
| [architecture/autonomy-ops-status-and-open-questions.md](architecture/autonomy-ops-status-and-open-questions.md) | Status ops + pytania otwarte (z proponowanymi odpowiedziami); baseline `5894906` |
| [architecture/intent-orchestration-and-fallbacks.md](architecture/intent-orchestration-and-fallbacks.md) | Intent packs, recipe policy, capability fallbacki, rola LLM (generycznie; publish tylko jako przykład) |
| [plans/autonomy-implementation-roadmap.md](plans/autonomy-implementation-roadmap.md) | Roadmapa: fazy 0–8 + kolejność jednostek zmian (PR table) |
| [autonomy-cli-runbook.md](autonomy-cli-runbook.md) | Runbook: NL z shella (`subactor` / `subactor-run`) → AQL/OQL/URI → Plesk sync; current vs target dla docs.subactor.com |
| [plans/docs-subactor-com-publish.md](plans/docs-subactor-com-publish.md) | Plan implementacji: allowlist `docs/`, recipe, NL → publish na docs.subactor.com |
| [plans/intent-capability-fallbacks.md](plans/intent-capability-fallbacks.md) | Skrót / pointer do dokumentu architektury intent + fallbacki |
| [deployment/docs-httpdocs-sync.urirun.json](deployment/docs-httpdocs-sync.urirun.json) | Recipe urirun (dry-run → apply) |

## Platforma (Organization OS)

| Dokument | Opis |
|----------|------|
| [platform/ORGANIZATION_OS_ARCHITECTURE.md](platform/ORGANIZATION_OS_ARCHITECTURE.md) | Warstwy AQL/OQL, Bridge, Control, LLM |
| [platform/BUSINESS_OPERATING_SYSTEM.md](platform/BUSINESS_OPERATING_SYSTEM.md) | Model operacyjny biznesu |
| [platform/TASK_PROCESS_RUNTIME.md](platform/TASK_PROCESS_RUNTIME.md) | Runtime zadań / URI Process |
| [platform/TESTQL_PROJECT_GATES.md](platform/TESTQL_PROJECT_GATES.md) | Bramki TestQL |
| [platform/CODEBASE_HEALTH.md](platform/CODEBASE_HEALTH.md) | **Stan kodu, source of truth, plan refaktoru (code2llm)** |

Szersza dokumentacja operacyjna: [`platform/docs/`](../platform/docs/) w repozytorium platformy (deploy assembly).

## Indeksy analizy kodu

Wygenerowane przez `code2llm` w `project/`:

- `analysis.toon.yaml` — HEALTH, REFACTOR, LAYERS  
- `map.toon.yaml` — mapa modułów  
- `evolution.toon.yaml` — kolejka splitów  
- `context.md` — narracja dla LLM  

Zasady skanowania i triażu mirrorów: [CODEBASE_HEALTH.md](platform/CODEBASE_HEALTH.md).
