---
{
  "schema": "subactor.doc/v1",
  "id": "docs.architecture.intent-contract-and-human-machine-source-of-truth-2026-07-22",
  "version": 4,
  "status": "current",
  "updated": "2026-07-22"
}
---

# Intent Contract — wspólne źródło prawdy człowieka i maszyny

## Werdykt

Subactor nie powinien dodawać kolejnego samodzielnego DSL wykonawczego. System
ma już rozdzielone języki i kontrakty dla authority, operacji, efektu,
wykonania oraz historii. Brakuje mu natomiast **wersjonowanej instancji
intencji**, która zachowuje to, co napisał człowiek lub maszyna, oraz oddziela
oryginalną wypowiedź od jej deterministycznej interpretacji.

Rekomendowanym rozwiązaniem jest `subactor.intent-contract/v1`: kanoniczny,
walidowany dokument JSON, renderowany ludziom jako formularz lub Markdown.
Kontrakt intencji nie wykonuje kodu i nie nadaje authority. Po akceptacji wiąże
się z Process Packiem, AQL, OQL, EQL, URI Process i ticketem Planfile.

## Ocena dwóch wariantów RULE-DSL

Przeanalizowano lokalne warianty:

- `wellmanifest/new-project/GPT56Luna/CONTRIBUTING.md`;
- `wellmanifest/new-project/Opus48Medium/CONTRIBUTING.md`.

Oba trafnie formalizują stany, priorytety, reguły `WHEN/THEN`, wymagane dowody,
bramki publikacji i warunki zakończenia. Wariant GPT56Luna jest dokładniejszą
maszyną stanów i dzieli reguły na moduły. Wariant Opus48Medium jest krótszy,
czytelniejszy i lepiej pokazuje precedencję polityk.

Nie powinny jednak zostać skopiowane jako źródło prawdy Subactora:

- są pseudojęzykiem bez wersjonowanego parsera, typów i kanonicznego AST;
- mieszają intencję, politykę repozytorium, stan środowiska i polecenia shell;
- literalne ścieżki i nazwy narzędzi szybko stają się nieaktualne;
- samo zdanie `DO RUN` nie jest dowodem authority, wykonania ani spełnienia
  postcondition;
- swobodny tekst maszyny mógłby niejawnie rozszerzyć dozwolone operacje.

Najbardziej wartościowe elementy tych dokumentów należy przejąć do schematu:
precedencję, jawne stany, reguły stop, wymagane evidence, Definition of Done oraz
rozdzielenie faktu potwierdzonego od deklaracji.

## Języki i kontrakty, które już istnieją

| Warstwa | Odpowiedzialność | Czego nie powinna przejmować |
|---|---|---|
| Intent Pack | rozpoznanie klasy intencji i typowanych slotów | pojedynczej wypowiedzi i jej akceptacji |
| Strategy DSL | wybór zatwierdzonej strategii i abstrakcyjnych capabilities | authority i dowolnego URI |
| AQL | kto ma authority, w jakim zakresie i czasie | szczegółowego planu wykonania |
| OQL | kanoniczna operacja i zależności | transportu i sekretów |
| URI Process | dokładny adres wykonania | decyzji biznesowej |
| EQL | oczekiwany rezultat i niezależna weryfikacja | uznawania samego HTTP 200 za sukces celu |
| Process Envelope v2 | komplet definicji procesu przypisany do ticketu | oryginalnego dialogu jako authority |
| SODL/1 | append-only zdarzenie, korelacja i kontrolowany replay | logiki biznesowej i planowania |
| Planfile | stan pracy, historia i completion receipt | niezatwierdzonej interpretacji LLM jako faktu |

Nie istnieje więc jeden brakujący „język wykonawczy”. Istnieje luka pomiędzy
surową wypowiedzią a wybranym, zatwierdzonym Process Packiem.

## Zaimplementowany rdzeń Intent Contract v1

Kanoniczna instancja zawiera:

```json
{
  "schema": "subactor.intent-contract/v1",
  "id": "INT-...",
  "version": 1,
  "status": "proposed",
  "statements": [
    {
      "actor": "human:founder",
      "kind": "request",
      "text": "...",
      "recorded_at": "...",
      "content_hash": "sha256:..."
    },
    {
      "actor": "bot:planner",
      "kind": "interpretation",
      "text": "...",
      "recorded_at": "...",
      "content_hash": "sha256:..."
    }
  ],
  "normalized": {
    "goal": "...",
    "constraints": [],
    "non_goals": [],
    "inputs": {},
    "expected_outcomes": [],
    "acceptance_criteria": []
  },
  "bindings": {
    "process_pack_ref": "...",
    "aql_ref": "...",
    "eql_refs": [],
    "capabilities": []
  },
  "decision": {
    "state": "pending",
    "accepted_by": null,
    "accepted_at": null,
    "contract_hash": "sha256:..."
  },
  "links": {
    "ticket_ids": [],
    "problem_id": null,
    "knowledge_refs": []
  }
}
```

Ostateczny wire contract znajduje się w
`contracts/schemas/intent-contract.schema.v1.json`, a deterministyczna
kanonikalizacja, hashowanie i reguły semantyczne w
`runtime/src/intent-contract.mjs`. URI, transport, vault entry i sekrety nie
mogą być generowane przez model. `capabilities` pochodzą z katalogu, a dokładne
URI dopiero z deterministycznego bindingu po AQL.

Runtime udostępnia również deterministyczne projekcje Markdown/form oraz
semantic diff. Projekcje zawsze wskazują `base_hash`; nie mogą zapisać zmiany,
zaakceptować intencji ani uruchomić procesu.

Control posiada teraz append-only Intent Registry. Udostępnia ograniczone
scope'ami API do utworzenia propozycji, dopisania nowej wersji z kontrolą
`base_hash`, pobrania wersji, projekcji Markdown/form i semantic diff. Rejestr
odrzuca statusy zaakceptowane i wykonawcze, podszycie autora, nieznane pola oraz
mutację wcześniejszej wersji. Nie istnieje endpoint `accept` ani `execute`.

`subactor.intent-binding/v1` formalizuje osobny, niemutowalny związek pomiędzy
hashem intencji, rewizją projektu i dokładnymi rewizjami Process Packa,
Strategy/AQL/OQL/EQL lub artefaktów. Binding nie zawiera URI wykonawczego i nie
nadaje authority. Obecnie jest kontraktem Schema + fixture; deterministyczny
binder i bramka akceptacji pozostają kolejnym etapem.

## Zasady współautorstwa człowieka i maszyny

1. Oryginalna wypowiedź jest niezmienna i zachowuje autora oraz hash.
2. Interpretacja maszyny jest osobnym statementem, nigdy podmianą wypowiedzi.
3. Zmiana znaczenia tworzy nową wersję kontraktu; wcześniejsza pozostaje
   audytowalna.
4. Człowiek może poprawić pola znormalizowane formularzem albo dodać kolejny
   statement. System pokazuje semantyczny diff przed akceptacją.
5. Akceptacja wiąże hash dokładnej wersji. Późniejsza zmiana unieważnia plan i
   wymaga ponownego preflightu.
6. LLM może proponować pack i sloty tylko z allowlisty. Nie tworzy authority,
   tras, odbiorców eskalacji ani nowych capabilities.
7. Ticket wykonawczy odwołuje się do niezmiennego `intent_ref`; zamknięcie
   ticketu nie może przepisać historycznej intencji.

## Lifecycle intencji

```text
draft → proposed → clarification_required → accepted → planned
      → executing → verified → closed
      ↘ rejected
      ↘ superseded
```

`verified` wymaga receipt EQL. `closed` oznacza osiągnięcie celu albo jawną
decyzję o zakończeniu. Sam merge, wysłanie wiadomości lub odpowiedź API nie
przesuwają kontraktu do `verified`.

## Problemy i równoległe naprawy

Intent Contract może wskazywać jeden `problem_id`, ale każda alternatywna
naprawa otrzymuje osobny `candidate_id` i ticket. Kandydaci dzielą oczekiwany
rezultat EQL, lecz zachowują własny plan, branch, evidence i receipt. Deduplikacja
dotyczy identycznych obserwacji albo ponownego wykonania tego samego kandydata,
nie różnych hipotez rozwiązania.

## Źródło prawdy

Kanonicznym zapisem powinien być walidowany JSON z hashem i wersją. Formularz,
Markdown, odpowiedź API oraz prompt LLM są projekcjami tego samego dokumentu.
Surowe statements są provenance, a zaakceptowana sekcja `normalized` jest
wiążącą intencją. AQL pozostaje jedynym źródłem authority, Planfile źródłem
stanu wykonania, a EQL receipt źródłem potwierdzonego rezultatu.

## Stan wdrożenia

- wykonane: schema, fixture, walidacja Runtime, canonical hash, projekcje,
  semantic diff, append-only registry oraz API `propose/revise/get/preview/diff`;
- wykonane kontraktowo: `subactor.intent-binding/v1` wiążący intencję, projekt i
  wersjonowane zależności governance;
- otwarte: akceptacja przez uprawnionego principal-a, deterministyczny binder,
  `intent_ref`/`intent_hash` w Process Envelope i Planfile oraz wspólny readiness
  hook blokujący wykonanie nieaktualnej intencji.

Szczegółowy plan implementacji znajduje się w
[`../plans/intent-contract-continuation-2026-07-22.md`](../plans/intent-contract-continuation-2026-07-22.md).
