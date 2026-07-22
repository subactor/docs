---
{
  "schema": "subactor.doc/v1",
  "id": "docs.architecture.adr.008-resource-lifecycle-control-plane",
  "version": 3,
  "status": "current",
  "updated": "2026-07-22"
}
---

# ADR-008: wspólny lifecycle zasobów control plane

- **Status:** Accepted
- **Data:** 2026-07-22
- **Kontrakt:** `subactor.resource-lifecycle/v1`
- **Implementacja:** `orchestrator/src/resource-lifecycle.mjs`
- **JSON Schema:** `orchestrator/schemas/resource-lifecycle.schema.v1.json`

## Kontekst

Tickety, artefakty tekstowe, witryny, rekordy DNS i domeny mają dziś różne
źródła prawdy oraz różne walidatory. Jest to poprawne na granicach domen, ale
utrudnia Orchestratorowi odpowiedź na wspólne pytania:

- jaki zasób istnieje i jaka jest jego stabilna tożsamość;
- gdzie znajduje się desired state, a gdzie observed state;
- czy dowody potwierdzają stan oczekiwany;
- kto ma obserwować, uzgadniać i reagować;
- czy kolejne przejście lifecycle jest dozwolone.

Samo skopiowanie wszystkich danych do jednej bazy stworzyłoby konkurencyjne
źródła prawdy i ryzyko działania na nieaktualnej projekcji.

## Decyzja

Orchestrator utrzymuje jeden wersjonowany **kontrakt projekcji lifecycle**, a nie
jedną fizyczną bazę wszystkich faktów. Każdy obiekt `subactor.resource-lifecycle/v1`
wiąże:

1. stabilną tożsamość i kanoniczny URI zasobu;
2. URI autorytetu desired state;
3. URI autorytetu observed state;
4. opcjonalny, ograniczony `mutation_uri`;
5. wymagane checks i evidence;
6. odpowiedzialności `owner`, `observer`, `reconciler`, `escalation`;
7. wyliczony stan i jedno następne działanie.

Stan lifecycle nie jest przyjmowany z odpowiedzi LLM. Jest deterministycznie
wyliczany z projekcji desired/observed i receipts walidacji.

### Autorytety per typ zasobu

| Typ | Tożsamość | Desired state | Observed state |
| --- | --- | --- | --- |
| `ticket` | `planfile://tickets/<id>` | Planfile envelope i inputs | Planfile status, historia i receipts |
| `text_artifact` | `artifact://subactor/<path>/r<revision>` | plik źródłowy w repozytorium | validation receipt Artifact Registry |
| `website` | `https://<host>/` | Site Resources / release manifest | publiczny HTTP, TLS i release marker |
| `dns_record` | `dns://<domain>/<type>/<name>` | deklaracja DNS w repozytorium | autorytatywny provider DNS / publiczny resolver |
| `domain` | `domain://<fqdn>` | inventory domen i kontrakt publikacji | registrar, provider DNS, TLS i publiczny endpoint |

Artifact Registry pozostaje deterministyczną projekcją plików tekstowych.
Planfile pozostaje źródłem stanu ticketów. Provider DNS pozostaje źródłem
obserwacji DNS. Vault pozostaje jedynym źródłem sekretów. Projekcja lifecycle
przechowuje wyłącznie odwołania i fingerprinty — nigdy credentiale.

## Stany i działania

Wspólne stany to `declared`, `ready`, `reconciling`, `verified`, `drifted`,
`blocked`, `failed` i `retired`.

| Stan | Następne działanie |
| --- | --- |
| `declared`, `ready`, `reconciling` | `observe` przez przypisanego obserwatora |
| `drifted` | `reconcile` przez właściciela zdolności |
| `blocked` | `notify` do wskazanego autorytetu lub człowieka |
| `failed` | utworzenie/deduplikacja ticketu u adresata eskalacji |
| `verified`, `retired` | `none` |

`verified` wymaga wszystkich obowiązkowych checks w stanie `pass` oraz co
najmniej jednego evidence URI. `blocked`, `failed` i `drifted` wymagają jawnych
reason codes. Lista dozwolonych przejść jest częścią wersjonowanego kontraktu.

## Zakres implementacji v1

Orchestrator udostępnia:

- walidator pojedynczego obiektu i przejścia;
- adaptery dla ticketu, artefaktu, WWW, rekordu DNS i domeny;
- zbiorczy snapshot z licznikami, wykryciem duplikatów i ograniczoną listą
  następnych działań;
- fail-closed kontrolę credentiali w projekcji;
- routing następnego działania według odpowiedzialności przypisanej do typu.
- observation-only CLI `subactor-resource-lifecycle`, które przyjmuje ograniczony
  JSON przez stdin i emituje snapshot bez wykonywania mutacji.

JSON Schema opisuje wire contract. Moduł Orchestratora jest implementacją reguł
semantycznych, których sam JSON Schema nie wyraża: derivation stanu, przejścia,
fingerprinty i zakaz sekretów.

## Konsekwencje

- Katalog lifecycle daje jeden widok kontrolny i wspólną walidację, ale nie
  zastępuje autorytetów domenowych.
- Mutacja pozostaje osobnym URI Process z AQL, exact plan hash, grantem i EQL.
  `next_action` nie jest sam w sobie uprawnieniem do apply.
- Każdy adapter musi zachować URI dowodu i czas obserwacji; brak obserwacji nie
  może zostać zamieniony na `verified`.
- Kanał powiadomienia jest wykonywany przez odpowiedzialnego bota/connector.
  Kontrakt wskazuje adresata, lecz nie omija polityki komunikacji ani HITL.

## Dalsza adopcja

1. Control ma publikować read-only snapshot z bieżących adapterów.
2. Ticket reconciler, public status probe i DNS observer mają emitować ten sam
   kontrakt zamiast własnych, częściowo zgodnych statusów.
3. `next_action` ma być przekładane na wersjonowany URI Process i receipt, a nie
   bezpośrednią mutację.
4. Status publiczny może konsumować bezpieczną projekcję WWW/domen, bez
   wewnętrznych reason codes, ścieżek repozytorium i danych autoryzacyjnych.
5. Po migracji należy usunąć duplikujące klasyfikatory stanów w adapterach,
   zachowując walidatory domenowe jako źródła checks.

## Dowody akceptacji

Testy Orchestratora obejmują wszystkie pięć adapterów, stan `verified`, drift
DNS, błąd HTTP, blokadę ticketu, routing odpowiedzialności, zakaz credentiali,
macierz przejść oraz zbiorczy snapshot.
