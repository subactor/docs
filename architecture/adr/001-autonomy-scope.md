# ADR-001: Zakres autonomii

- **Status:** Proposed  
- **Data:** 2026-07-18  
- **Kontekst:** [`../autonomy-recommended-solution.md`](../autonomy-recommended-solution.md) §2.1  
- **Pytanie statusowe:** „Scope dowolne zadanie vs katalog intent packs?”

## Decyzja

System autonomicznie wykonuje zadania z **wersjonowanego katalogu intent packów**.

LLM może:

- wybrać istniejący pack (`pack_id`),
- wypełnić dozwolone parametry (situation slots).

LLM **nie może**:

- tworzyć URI,
- wybierać transportów,
- wybierać identyfikatorów vault,
- definiować polityk wykonania (`on_fail`, timeout, retry).

„Dowolne zadanie” = dowolna kompozycja **zatwierdzonych** zdolności, nie dowolny kod ani operacja wymyślona przez model.

## Konsekwencje

- Intent pack = SSOT celu; AQL contract = SSOT autoryzacji (pack **nie** zawiera `ALLOW`).
- Nowy cel = nowy pack + review governance (HITL), nie free-form LLM.
- Align z [`../intent-orchestration-and-fallbacks.md`](../intent-orchestration-and-fallbacks.md).
