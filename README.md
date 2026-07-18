# docs.subactor.com

Publiczna dokumentacja Subactor.

## Autonomy (CLI → connectors)

| Dokument | Opis |
|----------|------|
| [autonomy-cli-runbook.md](autonomy-cli-runbook.md) | Runbook: NL z shella (`subactor` / `subactor-run`) → AQL/OQL/URI → Plesk sync; current vs target dla docs.subactor.com |

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
