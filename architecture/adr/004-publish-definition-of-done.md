# ADR-004: Definition of Done publikacji (verify)

- **Status:** Accepted  
- **Data:** 2026-07-18  
- **Kontekst:** [`../autonomy-recommended-solution.md`](../autonomy-recommended-solution.md) §2.4  
- **Pytanie statusowe:** „Monitoring / verify obowiązkowe?” — **rozstrzygnięte**  
- **Kryteria statusowe:** D6, D7 w [`../autonomy-ops-status-and-open-questions.md`](../autonomy-ops-status-and-open-questions.md)

## Decyzja

Dla intentu typu `publish` verify **nie** jest opcjonalnym ostrzeżeniem.

```text
upload OK + public verify FAIL
= applied_unverified
≠ sukces planu
→ rollback treści (ADR-005) lub ticket
```

`ok: true` / `status: completed` dopiero gdy łącznie:

1. DNS prowadzi do oczekiwanego celu (ADR-002) — observed == desired,
2. TLS SAN zawiera hostname,
3. HTTPS zwraca 200,
4. fingerprint treści = wdrożony release (np. `/__subactor_release.json`).

### CURRENT vs TARGET

| | |
| --- | --- |
| CURRENT | Dry-run + kill switch; public HTTPS docs może nadal być Pages |
| TARGET | Pełny DoD D1–D9 na Plesk origin |
| Nearest milestone | Mocked mutate + deny gates — **bez** twierdzenia o publicznym publish success |

## Konsekwencje

- API nie redukuje lifecycle do jednego boolean.
- Kryteria D1–D11 statusu pozostają mierzalne; D6/D7 stają się twardym DoD produkcyjnym.
- Nie deklarować sukcesu live HTTPS publish, gdy DNS nadal wskazuje GitHub Pages.
