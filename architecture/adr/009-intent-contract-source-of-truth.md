---
{
  "schema": "subactor.doc/v1",
  "id": "docs.architecture.adr.009-intent-contract-source-of-truth",
  "version": 1,
  "status": "current",
  "updated": "2026-07-22"
}
---

# ADR-009: Intent Contract jako źródło prawdy intencji

- **Status:** Accepted
- **Data:** 2026-07-22
- **Kontrakt:** `subactor.intent-contract/v1`
- **Schema:** `contracts/schemas/intent-contract.schema.v1.json`
- **Implementacja:** `runtime/src/intent-contract.mjs`

## Kontekst

Subactor rozdziela authority, operację, trasę wykonania, oczekiwany efekt i
stan pracy pomiędzy AQL, OQL, URI Process, EQL, Process Envelope oraz Planfile.
Brakowało kanonicznej instancji, która zachowuje surowe wypowiedzi człowieka i
maszyny, pokazuje ich interpretację oraz wiąże dokładnie zaakceptowane znaczenie.

Swobodny RULE-DSL lub prompt nie może pełnić tej funkcji: nie ma stabilnego AST,
może mieszać politykę z poleceniami wykonawczymi i nie stanowi authority.

## Decyzja

`subactor.intent-contract/v1` jest wersjonowaną kopertą JSON, a nie nowym
językiem wykonawczym. Kanoniczny dokument zawiera:

1. niezmienne statements z aktorem, czasem i hashem treści;
2. zaakceptowaną normalizację celu, ograniczeń, wejść i kryteriów akceptacji;
3. referencje do istniejących Process Packów, strategii, AQL, OQL i EQL;
4. decyzję związaną z hashem normalizacji i całego kontraktu;
5. referencje do ticketu, problemu, kandydata naprawy, wiedzy i receipts;
6. provenance poprzedniej wersji.

Klucze obiektów są sortowane przed serializacją, kolejność tablic jest
znacząca, a SHA-256 jest liczony z kanonicznego JSON. `contract_hash` nie jest
włączany do własnego preimage. Zmiana statementu, normalizacji, bindingu,
decyzji albo linku zmienia hash.

## Granice bezpieczeństwa

- Kontrakt nie nadaje authority i nie zastępuje AQL ani apply grantu.
- Nie przyjmuje dokładnych URI wykonawczych, poleceń, skryptów ani shella.
- Nie przechowuje sekretów ani wartości credentiali.
- LLM może zaproponować normalizację i referencję z allowlisty, ale nie może
  nadpisać statementu człowieka ani samodzielnie zaakceptować jego intencji.
- `executing` wymaga Process Packa, AQL, EQL i ticketu; `verified` wymaga receipt.
- JSON jest źródłem prawdy. Formularz i Markdown są deterministycznymi
  projekcjami, nie konkurencyjnymi dokumentami.

## Lifecycle

Dozwolone stany to `draft`, `proposed`, `clarification_required`, `accepted`,
`planned`, `executing`, `verified`, `closed`, `rejected` i `superseded`.
Zmiana znaczenia tworzy nową wersję z `previous_ref` i `previous_hash`.
Wcześniejsza wersja pozostaje audytowalna; nie jest przepisywana w miejscu.

## Konsekwencje

- Intent Contract zamyka lukę provenance pomiędzy dialogiem a ticketem.
- Planfile i Process Envelope muszą w kolejnym etapie zapisywać `intent_ref`
  oraz `intent_hash` i unieważniać niezgodny plan/grant.
- Potrzebny jest deterministyczny renderer formularza/Markdown i semantic diff.
- Równoległe naprawy mogą współdzielić `problem_id`, ale zachowują osobne
  `candidate_id`, plany, evidence i receipts.

## Dowody akceptacji

Testy Runtime sprawdzają stabilność kanonikalizacji, wykrywanie zmiany treści,
zakaz sekretów i wykonywalnych pól oraz wymagane bindingi lifecycle. Repozytorium
Contracts zawiera JSON Schema i kanoniczny pozytywny fixture.

