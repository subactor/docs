---
{
  "schema": "subactor.doc/v1",
  "id": "docs.architecture.system-overview",
  "version": 1,
  "status": "current",
  "updated": "2026-07-30"
}
---

# Subactor — całościowy opis systemu (jak czytać i jak działa)

Jedna strona startowa: **co to jest**, **jak działa kolejno**, **gdzie jest
dokumentacja toru**, **jak odróżnić zdrowy system od autonomii wykonawczej**.

Wersja knowledge: `knowledge://subactor/architecture.system-overview/v1`.

## 1. Jedno zdanie

Subactor to Organization OS z pętlą MAPE-K: obserwuje świat (twin, metryki,
receipts), analizuje sytuację (DOQL/DQL), planuje w katalogu zatwierdzonych
zdolności (intent / Strategy / remediation), wykonuje wyłącznie URI Process z
dry-run → `plan_hash` → grant → apply → EQL, a wiedzę trzyma w wersjonowanym
`knowledge://` i artifact registry. LLM proponuje packi i sloty — **nie** URI,
vault ani mutacje (ADR-001).

## 2. Warstwy (od człowieka do skutku)

```text
NL / Founder form / CLI
        │  require-llm → proposed DSL
        ▼
Intent Contract + Strategy DSL + AQL (authority)
        ▼
Planfile ticket (Task) + OQL + EQL envelope
        ▼
Control: lifecycle, queue controller, remediation, posture
        ▼
Bridge → URI Process → connectors (Plesk, DNS, mail, …)
        ▲                    │
        │   twin observe     │ dry-run / apply
        │   (ADR-012)        ▼
        └──── EQL read-back + SODL audit
```

| Warstwa | Rola | Nie jest |
| --- | --- | --- |
| Intent / Strategy | „co ma być prawdą” | zgodą na mutację |
| AQL | kto i w jakim zakresie | dowodem skutku |
| Twin / DOQL | sytuacja i preconditions | authority mutate |
| URI Process | exact recipe | swobodnym skryptem LLM |
| EQL | expected effect + read-back | intencją |
| Kill switches env | fail-closed execute | awarią usług |

## 3. Pętla MAPE-K w Subactor

| | Live (typowo observe-strong) | Docelowo w zakresie kontraktu |
| --- | --- | --- |
| **M**onitor | dashboard, probes, twin query | to samo + świeższe facts |
| **A**nalyze | DOQL, problem-reaction, research | research przed eskalacją zawsze |
| **P**lan | remediation, forms, strategy | automatic ready przy consumers=1 |
| **E**xecute | **wyłączone** bramkami (celowo) | dry-run→hash→grant→apply→EQL |
| **K**nowledge | knowledge://, artifacts | review_after + append-only |

`15/15 healthy` **≠** `autonomy_ready`. Zdrowy stack z
`AUTONOMOUS_QUEUE_CONSUMERS_ENABLED=0` i `AUTONOMY_MUTATIONS_ENABLED=0` to
**obserwacja silna, execute gated** — nie blackout.

## 4. Tor mutacji (kolejność twarda)

Kanonicznie:

[autonomy-execution-pipeline.md](./autonomy-execution-pipeline.md) ·
`knowledge://subactor/architecture.autonomy-execution-pipeline/v2`

```text
NL → DSL/intent → Task → URI (katalog) → Twin observe
  → payload validate → dry-run → plan_hash → grant
  → apply (env gate) → EQL / rollback
```

## 5. Kolejność czytania dokumentacji

### Zrozumieć system od zera

1. Ten dokument (overview)
2. [platform/ORGANIZATION_OS_ARCHITECTURE.md](../../platform/docs/ORGANIZATION_OS_ARCHITECTURE.md) — warstwy kontraktu
3. `knowledge://subactor/architecture.autonomy-definition/v1` — czym jest autonomia
4. [autonomy-execution-pipeline.md](./autonomy-execution-pipeline.md) — tor NL→EQL
5. [autonomy-loop-and-twin.md](./autonomy-loop-and-twin.md) — gdzie live się rwie
6. [autonomy-posture-authority.md](./autonomy-posture-authority.md) — decyzja Foundera vs bramki env
7. ADR: [001-autonomy-scope](./adr/001-autonomy-scope.md), [012-uri-twin](./adr/012-uri-twin-observe-layer.md)

### Operacje i runbook

8. [autonomy-cli-runbook.md](../autonomy-cli-runbook.md) — NL z shella
9. `knowledge://subactor/architecture.autonomy-architecture-blockers/v3` — bloki vs kill switch
10. [autonomy-recommended-solution.md](./autonomy-recommended-solution.md) — katalog zdolności, fazy

### API do „co się dzieje teraz”

| Endpoint | Pytanie |
| --- | --- |
| `GET /health` | safety mode z env (bez claimu Foundera) |
| `GET /api/system/dashboard` | usługi + tickety + `autonomy_ready` |
| `GET /api/autonomy/posture` | czy bramki = decyzja Foundera (DOQL) |
| `GET /api/autonomy/control` | **MAPE-K + ryzyka braku autonomii + rekomendacje kontroli** |

## 6. Ryzyka braku autonomii (klasy)

| Klasa | Znaczenie | Typowa kontrola |
| --- | --- | --- |
| `intentional_gate` | kill switch / mutation_gate | nie włączać mutacji „bo ticket czeka” |
| `observe_only_healthy` | stack OK, execute off | posture form → bounded execute |
| `posture_drift` | decyzja ≠ live env | reconcile compose/env z ticketem |
| `structural_gap` | brak trasy / coding agent | research + Process Pack |
| `lifecycle_stall` | blocked-by / dependency | recheck child terminal |
| `human_boundary` | sekret, grant, polityka | form/mail HITL |

Powierzchnia kontroli (`/api/autonomy/control`) rozdziela te klasy, żeby operator
nie leczył fail-closed jak awarii.

## 7. Decyzje nienegocjowalne

1. LLM nie inventuje URI, vault id, transportu (ADR-001).
2. Twin nie mutuje (ADR-012).
3. Mutacja: dry-run → exact `plan_hash` → single-use grant → env gate → EQL.
4. Founder tylko na granicach POLICY — nie na każdym observe.
5. Knowledge append-only by version; brak tokenów w URL/query/log.

## 8. Powiązane knowledge

- `knowledge://subactor/architecture.system-overview/v1` (ten przegląd)
- `knowledge://subactor/architecture.autonomy-definition/v1`
- `knowledge://subactor/architecture.autonomy-execution-pipeline/v2`
- `knowledge://subactor/architecture.autonomy-architecture-blockers/v3`
- `knowledge://subactor/architecture.uri-twin-scope/v3`
