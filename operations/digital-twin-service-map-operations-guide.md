---
{
  "schema": "subactor.doc/v1",
  "id": "docs.operations.digital-twin-service-map-operations-guide",
  "version": 1,
  "status": "current",
  "updated": "2026-07-24"
}
---

# Przewodnik Operacyjny: Digital Twin Service Map & Interaktywne Formularze Control

## Wprowadzenie

Niniejszy przewodnik opisuje procedury operacyjne i diagnostykę dla nowo wdrożonych komponentów pętli autonomii Subactor: mapy usług **Digital Twin** oraz **interaktywnych formularzy sterujących w panelu Control**.

---

## 1. Weryfikacja Mapy Usług Digital Twin

Uruchomienie weryfikacji sytuacji dla publicznych projektów odbywa się z poziomu repozytorium `platform`:

```bash
cd platform
node scripts/run-public-site-service-map.mjs
```

### Oczekiwany wynik

```text
=== Subactor Digital Twin Service Map Runner ===
Evaluating Intent: INT-FOUNDER-PUBLIC-SITE-001 (founder.subactor.com)
DOQL Service Map Valid: true
DOQL Capability Inventory Valid: true
Plesk Inventory Domain Role: undefined
DOQL Evaluation Summary:
  Profile ID: undefined
  Aggregates: undefined
  Decision Candidates: 0
```

---

## 2. Audyt Ticketów Planfile (`ticket-auditor.mjs`)

W celu weryfikacji spójności i otwartych prac należy uruchomić:

```bash
cd platform
node scripts/ticket-auditor.mjs --limit 20
```

Wszystkie otwarte zgłoszenia są poddawane walidacji w oparciu o aktualne blokery i wpisy w bazie wiedzy (`subactor.ticket-llm-context/v2`).

---

## 3. Utrzymanie Spójności Artefaktów i Próby CI

Przed wysłaniem zmian do repozytorium należy upewnić się, że rejestr artefaktów i weryfikacja DSL są czyste:

```bash
cd platform
npm run artifacts:build
npm run artifacts:check
npm run dsl:check
npm test
```

Wszystkie 730+ testów powinno zakończyć się wynikiem **PASS**.
