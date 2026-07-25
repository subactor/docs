---
{
  "schema": "subactor.doc/v1",
  "id": "docs.architecture.autonomy-architecture-blockers",
  "version": 2,
  "status": "current",
  "updated": "2026-07-25"
}
---

# Błędy architektury, które utrudniają autonomię

**Status:** inventarz wewnętrzny v2 (2026-07-25)  
**SSOT:** [`knowledge://subactor/architecture.autonomy-architecture-blockers/v2`](../../platform/config/knowledge/entries/architecture.autonomy-architecture-blockers.v2.md)  
**Definicja autonomii:** [`knowledge://subactor/architecture.autonomy-definition/v1`](../../platform/config/knowledge/entries/architecture.autonomy-definition.v1.md)

Autonomię hamuje nie „brak LLM”, lecz architektura, w której system nie domyka
obserwacja → decyzja w POLICY → wykonanie → verify — albo budzi Foundera na
fałszywym obrazie rzeczywistości.

## Klastry (priorytet)

| Klaster | Przykłady | Dotkliwość | Stan |
| --- | --- | --- | --- |
| **A. Prawda świata** | Plesk vs Pages; TLS bez DNS; false-reality PLF-884; równoległe remediacje; stale tunnel envelope | wysoka | otwarte / częściowo |
| **B. Kolejka / HITL** | bounce ready→waiting_input; ops human_boundary; duplikaty producenta; 50 waiting_input; exact_route→Founder | wysoka | otwarte / częściowo |
| **C. SSOT wykonania** | dual-run + Planfile; CLI `done` bez receiptów; Planfile timeout; stale step-catalog | wysoka–średnia | częściowo (PR10) |
| **D. Analyze/Repair** | diagnose≠repair pack; DOQL stub; planner bez tras; KPI notify≠execute | wysoka–średnia | otwarte |
| **E. Governance/kanały** | mutations=0 (celowe); child grant; `.env` SSOT; Founder 404; e-mail loop | średnia | celowe / otwarte |

## Co naprawiać najpierw

1. False-reality + łańcuch przyczynowy remediacji (nie TLS przed DNS).  
2. Jedna prawda publiczna (PR9).  
3. Pack→Planfile + receipts tylko z kolejki (PR10).  
4. Executable repair + typed DOQL.  
5. Dopiero NL/LLM i ewolucja.

## Sygnał operacyjny

`autonomy_ready=false` przy **15/15 healthy** oznacza backlog uwagi człowieka,
nie awarię stacku (`SYSTEM_STATE_2026-07-24`). Nie optymalizuj pod liczbę cykli
kontrolera ani ticketów.

## Powiązane

| Dokument | Rola |
| --- | --- |
| [`../operations/false-reality-and-shell-verification-2026-07-24.md`](../operations/false-reality-and-shell-verification-2026-07-24.md) | Ticket ≠ live |
| [`../../platform/docs/SYSTEM_STATE_2026-07-24.md`](../../platform/docs/SYSTEM_STATE_2026-07-24.md) | Głód waiting_input, duplikaty |
| [`unresolved-live-autonomy-blockers-2026-07-19.md`](./unresolved-live-autonomy-blockers-2026-07-19.md) | Live blockers |
| [`../operations/plesk-catalog-ticket-audit-2026-07-24.md`](../operations/plesk-catalog-ticket-audit-2026-07-24.md) | CLI≠queue, human_boundary ops |
| [`autonomy-ops-status-and-open-questions.md`](./autonomy-ops-status-and-open-questions.md) | PR9/PR10 |
| [`adr/003-approval-hitl-model.md`](./adr/003-approval-hitl-model.md) | Kiedy HITL jest OK |

*Bez sekretów. Review wiedzy: 2026-08-08.*
