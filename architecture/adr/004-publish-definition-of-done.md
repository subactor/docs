# ADR-004: Definition of Done publikacji (verify)

- **Status:** Proposed  
- **Data:** 2026-07-18  
- **Kontekst:** [`../autonomy-recommended-solution.md`](../autonomy-recommended-solution.md) §2.4  
- **Pytanie statusowe:** „Monitoring / verify obowiązkowe?”  
- **Kryteria statusowe:** D6, D7 w [`../autonomy-ops-status-and-open-questions.md`](../autonomy-ops-status-and-open-questions.md)

## Decyzja

Dla intentu typu `publish` verify **nie** jest opcjonalnym ostrzeżeniem.

```text
upload OK + public verify FAIL
= applied_unverified
≠ sukces planu
→ rollback lub ticket
```

`ok: true` / `status: completed` dopiero gdy łącznie:

1. DNS prowadzi do oczekiwanego celu (ADR-002),
2. TLS SAN zawiera hostname,
3. HTTPS zwraca 200,
4. fingerprint treści = wdrożony release (np. `/__subactor_release.json`).

## Konsekwencje

- API nie redukuje lifecycle do jednego boolean.
- Kryteria D1–D11 statusu pozostają mierzalne; D6/D7 stają się twardym DoD.
