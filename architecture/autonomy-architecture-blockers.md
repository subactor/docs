---
{
  "schema": "subactor.doc/v1",
  "id": "docs.architecture.autonomy-architecture-blockers",
  "version": 1,
  "status": "current",
  "updated": "2026-07-25"
}
---

# Błędy architektury, które utrudniają autonomię

**Status:** inventarz wewnętrzny (2026-07-25)  
**SSOT dla agentów:** [`knowledge://subactor/architecture.autonomy-architecture-blockers/v1`](../../platform/config/knowledge/entries/architecture.autonomy-architecture-blockers.v1.md)  
**Definicja autonomii:** [`knowledge://subactor/architecture.autonomy-definition/v1`](../../platform/config/knowledge/entries/architecture.autonomy-definition.v1.md)

Autonomię hamuje nie „brak LLM”, lecz architektura, w której system nie może
samodzielnie domknąć pętli obserwacja → decyzja w POLICY → wykonanie → verify
bez ręcznego klejenia albo fałszywego budzenia Foundera.

## Skrót (priorytet)

| # | Problem | Dotkliwość | Stan | Skutek dla autonomii |
| --- | --- | --- | --- | --- |
| 1 | Podwójna prawda DNS/treści (Plesk vs GitHub Pages) | wysoka | otwarte (PR9) | Apply „OK”, świat widzi starą treść; DoD HTTPS pada |
| 2 | TLS/ACME bez publicznego DNS (fałszywy origin probe) | wysoka | częściowo | Self-heal certu w złym momencie / zła diagnoza |
| 3 | Fałszywe `human_boundary` + bounce `ready→waiting_input` | wysoka | częściowo | Kolejka stoi, UI udaje postęp (PLF-898) |
| 4 | Ops blocker na Founderze zamiast botu | wysoka | częściowo | HITL na rutynowy DNS/TLS/route |
| 5 | Drift pack ↔ Planfile; dual-run shadow | średnia | częściowo (PR10) | Dwa SSOT tego samego celu |
| 6 | NL publish bez pełnego preflight infra | średnia | otwarte | Pętla urywa się na domenie/TLS/vault |
| 7 | Secrets/config poza pętlą (`.env` bez restartu) | średnia | otwarte | Urywa się na credential; fałszywy apply config |
| 8 | Publiczny ingress Foundera niegotowy | średnia | otwarte | HITL nie ma kanału z zewnątrz |
| 9 | DOQL/twin niepełny (stub query, brak adapters) | średnia | otwarte | Decyzje na niepełnej sytuacji / false-reality |
| 10 | Eskalacje bez stabilnego dedupe | średnia | częściowo | Spam ticketów Foundera |
| 11 | Master kill `AUTONOMY_MUTATIONS=0` | stan | celowe | Zero-touch mutate wyłączony (nie mylić z bugiem) |
| 12 | Brak pełnego process packu Problem→EQL | średnia | otwarte | Widzi błąd, nie zawsze materializuje self-heal |

## Co naprawiać najpierw

1. **Jedna prawda publiczna** — DNS→Plesk + verify ladder (PR9), bez uznawania Pages za zdrowe LKG.
2. **Prawdziwa granica człowieka** — bot na ops; Founder tylko sekret / `plan_hash` / konflikt POLICY; usuń bounce bez blokera.
3. **Jeden SSOT intencji** — Planfile z packa, dual-run `off` (PR10).
4. **Pełna sytuacja** — typed DOQL + reality-check zanim eskalacja.

LLM, nowe packi i self-evolution **nie** zastąpią punktów 1–3.

## Powiązane

| Dokument | Rola |
| --- | --- |
| [`autonomy-ops-status-and-open-questions.md`](./autonomy-ops-status-and-open-questions.md) | Problemy ops + pytania |
| [`autonomy-implementation-status.md`](./autonomy-implementation-status.md) | CURRENT / TARGET / LEGACY |
| [`autonomy-and-founder-communication-report-2026-07-20.md`](./autonomy-and-founder-communication-report-2026-07-20.md) | Backlog waiting_input, kanały |
| [`adr/002-dns-ssot.md`](./adr/002-dns-ssot.md) | DNS SSOT |
| [`adr/003-approval-hitl-model.md`](./adr/003-approval-hitl-model.md) | Kiedy HITL jest zamierzony |
| [`../plans/autonomy-implementation-roadmap.md`](../plans/autonomy-implementation-roadmap.md) | Kolejność PR |

*Bez sekretów. Review wiedzy: 2026-08-08.*
