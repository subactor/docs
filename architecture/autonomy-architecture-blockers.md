---
{
  "schema": "subactor.doc/v1",
  "id": "docs.architecture.autonomy-architecture-blockers",
  "version": 4,
  "status": "current",
  "updated": "2026-07-30"
}
---

# Błędy architektury, które utrudniają autonomię

**Status:** inventarz wewnętrzny v3 (2026-07-30)

**SSOT:** [`knowledge://subactor/architecture.autonomy-architecture-blockers/v3`](../../platform/config/knowledge/entries/architecture.autonomy-architecture-blockers.v3.md)
**Definicja autonomii:** [`knowledge://subactor/architecture.autonomy-definition/v1`](../../platform/config/knowledge/entries/architecture.autonomy-definition.v1.md)

Autonomię hamuje nie „brak LLM”, lecz architektura, w której system nie domyka
obserwacja → decyzja w POLICY → wykonanie → verify — albo budzi Foundera na
fałszywym obrazie rzeczywistości.

**Live 2026-07-30:** 15/15 healthy, `autonomy_ready=false`, consumers/mutations
off (celowe). Naprawiono: stale `blocked-by` po done form, brak propagacji
`human-baseline-review`, false connection `handler:codex`.

## Klastry (priorytet)

| Klaster | Przykłady | Dotkliwość | Stan |
| --- | --- | --- | --- |
| **A. Prawda świata** | Plesk vs Pages; TLS bez DNS; false-reality PLF-884; równoległe remediacje; stale tunnel envelope | wysoka | otwarte / częściowo |
| **B. Kolejka / HITL** | bounce ready→waiting_input; ops human_boundary; duplikaty producenta; 22 waiting_input; exact_route→Founder; ~~stale blocked-by after done form~~ | wysoka | częściowo (v3) |
| **C. SSOT wykonania** | dual-run + Planfile; CLI `done` bez receiptów; Planfile timeout; stale step-catalog; ~~codex/autonomy false resume~~ | wysoka–średnia | częściowo (PR10 + v3) |
| **D. Analyze/Repair** | diagnose≠repair pack; DOQL stub; planner bez tras; KPI notify≠execute; brak coding-agent consumer | wysoka–średnia | otwarte |
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
