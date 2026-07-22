---
{
  "schema": "subactor.doc/v1",
  "id": "docs.architecture.dql-autodiagnostics-2026-07-22",
  "version": 1,
  "status": "current",
  "updated": "2026-07-22"
}
---

# DQL/1 — deterministyczna autodiagnostyka Subactora

## Decyzja

DQL/1 (`subactor.diagnostic-profile/v1`) jest deklaratywnym profilem diagnozy
nad snapshotem systemu. Używa bezpiecznych wyrażeń TestQL oraz typowanych
operatorów dla kolekcji, grafów zależności i immutable artifact refs. Nie jest
językiem naprawy i nie wykonuje URI Process.

Kanonicznym źródłem prawdy jest JSON zgodny z
`contracts/schemas/diagnostic-profile.schema.v1.json`. Plik `.dql.json`, raport
Markdown lub panel są wyłącznie projekcjami tego AST.

## Zaimplementowany pionowy przekrój

- Contracts zawiera schema i kanoniczny fixture aktywnego profilu.
- Core zawiera ścisły, deterministyczny evaluator DQL.
- Platform posiada aktywny profil `control.architecture-integrity`.
- Read-only collector pobiera rzeczywisty kod tras Intent API, graf kroków
  Process Packa reconciliacji oraz immutable revision refs z Artifact Registry.
- `npm run diagnostics:check` emituje `subactor.diagnostic-report/v1` i kończy
  się błędem, gdy choć jeden inwariant jest naruszony.
- `test:meta` uruchamia diagnozę jako bramkę CI.

Pierwszy profil sprawdza:

1. czy proposal API intencji nie ma literalnej trasy `accept` lub `execute`;
2. czy graf procesu reconciliacji ma unikalne kroki, kompletne zależności i nie
   zawiera cyklu;
3. czy zależności governance wskazują immutable URI Artifact Registry.

## Kontrakt bezpieczeństwa

Każdy profil musi mieć `read_only=true` i
`automatic_mutation_allowed=false`. Finding zawiera stabilny fingerprint,
owner, Problem Profile candidate, opcjonalną strategię naprawy oraz wymagany
EQL. Jednocześnie zawsze deklaruje wymaganie ticketu, AQL i EQL przed przyszłą
reakcją wykonawczą.

LLM może zaproponować profil lub hipotezę, lecz parser odrzuca nieznane pola, w
tym `command`, `shell` i wykonywalne URI. Brak wymaganego snapshotu jest osobnym
problemem zależności, a nie fałszywie zielonym wynikiem.

## Relacja do istniejących języków

| Warstwa | Odpowiedzialność |
|---|---|
| DQL | porównanie obserwacji z inwariantem i utworzenie findingu |
| TestQL | bezpieczne wyrażenia logiczne wewnątrz inwariantu |
| EQL | niezależna walidacja rezultatu po ewentualnej naprawie |
| SODL | docelowy dziennik snapshotów, findingów i korelacji |
| AQL | authority Doctor/Repair/Validator |
| URI Process | sonda lub naprawa uruchamiana poza DQL |
| Planfile | lifecycle diagnozy i ticketów naprawczych |

## Uczciwe ograniczenia v1

- obserwacja granicy API wykrywa literalne trasy w źródle; nie korzysta jeszcze
  z runtime route registry;
- aktywny jest jeden profil i jeden collector;
- brak operatorów czasowych nad SODL, freshness i okien zdarzeń;
- raport nie jest jeszcze automatycznie zapisywany jako Problem Reaction ani
  nie tworzy ticketu;
- DQL nie wybiera RepairCandidate i nie uruchamia repair-agent ani validatora.

Plan dalszej integracji znajduje się w
[`../plans/dql-autodiagnostics-continuation-2026-07-22.md`](../plans/dql-autodiagnostics-continuation-2026-07-22.md).
