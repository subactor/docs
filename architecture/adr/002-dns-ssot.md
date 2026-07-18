# ADR-002: DNS jako SSOT

- **Status:** Proposed  
- **Data:** 2026-07-18  
- **Kontekst:** [`../autonomy-recommended-solution.md`](../autonomy-recommended-solution.md) §2.2  
- **Pytanie statusowe:** „Gdzie żyje prawda DNS/domen?” / „GitHub Pages vs Plesk?”

## Decyzja

| Warstwa | Rola |
| --- | --- |
| Repozytorium | **desired state** DNS |
| Provider DNS | **observed state** |
| Connector DNS | reconcile |
| Plesk | odbiorca konfiguracji — **nie** SSOT DNS |

Dla dokumentacji publicznej:

```text
docs.subactor.com → Plesk
```

Jeśli kiedyś docs miałyby zostać na GitHub Pages, intent „docs → Plesk” musi zniknąć z publicznego procesu. Dwie równoległe „prawdy” są źródłem obecnego mismatchu (Pages vs Plesk).

## Konsekwencje

- Migracja DNS to operacja **boundary-class** (HITL / ADR-003).
- Publiczny verify (ADR-004) obejmuje zgodność DNS z desired state.
- Rollback DNS do poprzedniego desired state (Pages) jest częścią planu awaryjnego (ADR-005).
