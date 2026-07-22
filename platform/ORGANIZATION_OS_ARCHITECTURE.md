---
{
  "schema": "subactor.doc/v1",
  "id": "docs.platform.organization.os.architecture",
  "version": 2,
  "status": "current",
  "updated": "2026-07-22"
}
---

# Architektura Subactor Organization OS

## Workspace i źródła prawdy

Lokalny workspace składa się z kanonicznych komponentów (`core`, `agents`,
`connectors`, `runtime`, `contracts`, …) oraz assembly `platform/` z mirrorami
w `platform/components/*`. Logikę komponentu zmienia się w repozytorium
kanonicznym i synchronizuje z mirrorem używanym przez testy Platformy.

Stan organizacji, pracy i dokumentacji nie jest przechowywany w jednym
nieformalnym pliku:

- Organization Core przechowuje rekordy domenowe;
- Planfile przechowuje tickety, zależności i receipts;
- Knowledge Base przechowuje wersjonowane założenia operacyjne;
- Artifact Registry przechowuje format, schemat i rewizję tekstowych artefaktów.

## Przepływ kontrolowany kontraktami

```text
 człowiek / maszyna
         |
         v
 +-------------------+       +--------------------------+
 | Intent Contract   |------>| Knowledge + Artifact     |
 | cel i ograniczenia|       | Registry (wersje, schema)|
 +-------------------+       +--------------------------+
         |
         v
 +------------------------------------------------------------------+
 | CONTROL PLANE                                                    |
 | Strategy DSL | AQL | DOQL/Digital Twin | OQL | lifecycle policy |
 | wybór        | auth| ocena sytuacji     | plan| retry/escalation |
 +------------------------------------------------------------------+
         |                       ^
         v                       | read model
 +-------------------+           |
 | Planfile          |<----------+
 | ticket + envelope |
 +-------------------+
         |
         v
 +-------------------+      +------------------+      +----------------+
 | Bridge guards     |----->| URI runtime /    |----->| Org Core, DNS, |
 | ticket-first, AQL |      | connectors       |      | SMTP, Plesk... |
 +-------------------+      +------------------+      +----------------+
         ^                                                    |
         +---------- EQL read-back + process receipt <--------+
                         |
                         v
              SODL audit + Planfile evidence

 LLM Gateway: typowana propozycja dla Control; nigdy skrót do Bridge.
```

Każda warstwa ma jedną odpowiedzialność. Intent Contract utrwala zaakceptowany
cel, Strategy DSL wybiera strategię, AQL nadaje authority, DOQL sumuje Digital
Twin, OQL planuje operację, URI adresuje wykonawcę, EQL sprawdza rezultat,
Process Envelope v2 wiąże kontrakty z ticketem, SODL zapisuje historię, a
Planfile utrzymuje stan pracy.

## Autonomiczny lifecycle ticketu

```text
 ticket open
    |
    +-- bot + komplet kontraktów? -- no --> blocked / owner queue
    |                 |
    |                yes
    |                 v
    |       claim -> running -> Bridge -> URI receipts
    |                                      |
    |                                      v
    |                            EQL read-back green?
    |                               | yes       | no
    |                               v           v
    |                             done     Problem Profile
    |                                           |
    |                                     reaction ticket
    |
    `-- human --> waiting_input --> responsibility lifecycle
```

Claim jest ponownie walidowany przed zmianą stanu. HTTP 2xx nie jest dowodem
ukończenia: potrzebne są receipts kroków, postcondition EQL i `verified_by`.

## Lifecycle odpowiedzialności Foundera

```text
 source ticket: founder + waiting_input
                  |
                  v
        klasyfikacja ważności
           |             |
        digest       natychmiastowy Process Pack v2
                           |
                  communications-bot -> Bridge -> SMTP
                           |
                 +---------+----------+
                 |                    |
           transport fail         delivered
                 |                    |
       delivery retry budget     response deadline
                 |                    |
                 +------ retry <-----+
                           |
                   nadal waiting_input?
                    | no         | tak, limit
                    v            v
                   stop    Planfile + aktywny digest
                           durable fallback receipt
```

Budżet awarii transportu i budżet skutecznie dostarczonej korespondencji są
oddzielne. Control narzuca politykę; `communications-bot` wykonuje wersjonowany
Process Pack. Zewnętrzny kanał jest dostępny dopiero po związaniu principalu,
odbiorcy, AQL, live preflightu i receiptów.

## Idempotencja instancji procesu

```text
 source PLF
   +-- attempt A: correlation=A -> key source:A -> execution A -> receipt A
   |                    retry A --------^ (ten sam skutek)
   `-- attempt B: correlation=B -> key source:B -> execution B -> receipt B
```

Idempotencja zapobiega podwójnemu skutkowi tej samej instancji. Jawnie
dozwolone ponowienie ma nowy ticket i nowy `correlation_id`.

## Granice domen

```text
 People Operations      ludzie, zespoły, onboarding
 Customer Operations    klienci, kontakty, relacje
 Communication          rozmowy, wiadomości, zobowiązania
 Contract Operations    umowy i wersje
 Project Operations     projekty, zadania, blokery
 Strategy Operations    decyzje i realizacja strategii
```

## Project Business Layer

```text
 hr-control
   |-- project-importer
   |     |-- website / directory / git importer
   |     |-- Markdown normalizer + deterministic blueprint
   |     `-- import TestQL + optional bounded LLM analysis
   |-- project reconciliation controller
   |     `-- source / legal / DNS / TLS / Plesk / content observation
   |-- Planfile + Process Pack registry
   `-- Organization Core
         `-- project workspace + Digital Twin projections
```

Publikacja lub inna mutacja infrastruktury wymaga authority, zielonych
preflightów, dokładnej trasy URI oraz postflight EQL.

## Control service — aktualne moduły

| Moduł | Odpowiedzialność |
|---|---|
| `delegation-manager.mjs` | wybór aktora i preview delegacji |
| `access-registry.mjs` | aktorzy i kontrakty AQL |
| `autonomous-queue-controller.mjs` | gotowość, claim i wykonanie kolejki botów |
| `founder-attention-policy.mjs` | klasyfikacja, deadline i budżety retry |
| `founder-attention-fallback.mjs` | trwały fallback Planfile/digest |
| `process-pack-registry.mjs` | wybór, kompilacja i walidacja wersji packów |
| `knowledge-base.mjs` | wersjonowany kontekst wewnętrzny |
| `artifact-registry.mjs` | projekcje i receipts artefaktów tekstowych |
| `system-dashboard.mjs` | zagregowany read model zdrowia systemu |
| `server.mjs` | HTTP, scheduler i integracja modułów |

## Aktualny stan i ograniczenia

- retry korespondencji Foundera i fallback Planfile/digest działają live;
- Process Pack v2 wiąże idempotencję z instancją procesu;
- SMTP receipts potwierdzono dla PLF-1033, PLF-1035 i PLF-1037;
- publiczna akcja Foundera pozostaje fail-closed do rozwiązania PLF-592;
- Signal, Slack, Teams i inne kanały wymagają formalnego mapowania principalu;
- LLM nie wybiera odbiorcy, nie tworzy URI i nie rozszerza budżetu prób.
