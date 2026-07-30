---
{
  "schema": "subactor.doc/v1",
  "id": "docs.plans.project-continuation-2026-07-30",
  "version": 4,
  "status": "current",
  "updated": "2026-07-29"
}
---

# Plan kontynuacji projektów — 2026-07-30

## Korekta architektoniczna: decyzja zawsze z NL → LLM → DSL

Semantyczna decyzja nie może pochodzić z heurystyki Control. Źródłem jest NL
człowieka, które LLM w trybie `require-llm` przekształca w typowany,
wersjonowany DSL. Wynik LLM ma status `proposed`; człowiek akceptuje jego
canonical hash albo wcześniej nadaje ograniczony policy grant dla tej klasy
decyzji.

Deterministyczny kod pozostaje potrzebny, ale tylko do walidacji schematu,
provenance, hashy, AQL, exact URI, EQL i receiptów. Nie interpretuje wypowiedzi,
nie wybiera intencji i nie uzupełnia znaczenia regexami, słowami kluczowymi,
scoringiem ani tabelą wyjątków. Gdy LLM jest niedostępny lub zwraca
niepoprawny structured output, decyzja nie powstaje i proces zatrzymuje się
fail-closed.

Wzorzec implementacyjny do przeniesienia z
`/home/tom/github/semcod/todo2code`:

- tryb `require-llm` bez semantycznego fallbacku;
- strict structured output walidowany względem JSON Schema;
- źródłowe NL, model, response ID i konfiguracja bez sekretów w audycie;
- wymuszone `status=proposed`, niezależnie od odpowiedzi modelu;
- stabilne ID i canonical hash wyliczane przez runtime;
- odrzucenie całej odpowiedzi przy błędnym cytowaniu, zależności lub schemacie.

## Cel i zakres

Ten dokument jest operacyjnym handoffem na 30 lipca 2026. Zbiera prace, które
zostały rozpoczęte, ale nie mają jeszcze pełnego łańcucha:

```text
implementacja → wdrożenie → wykonanie kontrolowane → niezależny read-back → EQL receipt
```

Nie zastępuje istniejących roadmap. Szczegóły architektury i długoterminowe
fazy pozostają w:

- [autonomy-implementation-roadmap.md](./autonomy-implementation-roadmap.md);
- [uri-twin-plesk-implementation-roadmap-2026-07-25.md](./uri-twin-plesk-implementation-roadmap-2026-07-25.md);
- [refactoring-status-and-roadmap-2026-07-29.md](./refactoring-status-and-roadmap-2026-07-29.md);
- [AUTONOMOUS_CONTROLLER_CONTINUATION_PLAN.md](../../platform/docs/AUTONOMOUS_CONTROLLER_CONTINUATION_PLAN.md).

## Zweryfikowany punkt startowy

Stan Control odczytany 2026-07-29 po wdrożeniu bieżących zmian:

| Wskaźnik | Stan |
| --- | ---: |
| Aktywne tickety | 32 |
| Aktywne tickety w kolejkach automatycznych | 20 |
| Gotowe do automatycznego wykonania | 0 |
| Wykonywane procesy ciągłe/testowe | 2 |
| Automatyczne, ale oczekujące | 18 |
| Projekty z aktywnym blockerem | 6 |
| Aktywna czasowa zgoda na mutacje | nie |

Dwa tickety `running` to proces ciągły Project Reconciliation Controller
(`PLF-613`) i guarded process testowego Connector LAN (`PLF-1330`). Nie są one
dowodem, że kolejka potrafi obecnie samodzielnie doprowadzić naprawę
produkcyjną do końca.

System działa autonomicznie w zakresie obserwacji, diagnozy, tworzenia planów,
przypomnień i fail-closed blokowania. Nie ma obecnie ani jednego zwykłego
ticketu spełniającego wszystkie warunki wykonania. Nie należy opisywać tego
stanu jako pełnej autonomii produkcyjnej.

## Co wykonano 29 lipca i czego jutro nie powtarzać

- Dodano dedykowaną komendę NL `subactor founder`, ale jej bieżący routing
  deterministyczny jest rozwiązaniem przejściowym i nie spełnia docelowej
  zasady NL → LLM → DSL.
- Rozszerzono projekcję statusu o liczbę ticketów automatycznych gotowych,
  wykonywanych i oczekujących oraz o ograniczoną listę najważniejszych akcji
  Foundera.
- Przebudowano usługę Control i potwierdzono, że wdrożony moduł odpowiada
  wersji z workspace.
- Wykonano bezpieczny pojedynczy cykl kontrolera: 52 zeskanowane tickety,
  30 rozważonych, 0 wykonalnych, 30 notyfikacji i 22 pominięte.
- Czasowa zgoda na mutacje została po teście cofnięta. Bramki pozostają
  zamknięte.
- Zielone były zestawy testów Founder CLI (11), Founder chat/autonomy (16),
  Control dla bieżącego zakresu (67), runtime (183) i
  `@subactor/planfile-orchestration` (33).

Zmiany Founder CLI i część zmian Control nadal znajdują się w brudnych,
współdzielonych worktree. Są wdrożone do bieżącego środowiska, ale nie stanowią
jeszcze odseparowanego, audytowalnego wydania.

## Wynik 30 lipca — slice 1: Founder NL w trybie `require-llm`

Pierwszy zakres P0 został wdrożony i zweryfikowany bez otwierania bramek
mutacji:

- `subactor founder` deklaruje `interpretation_mode=require-llm`;
- Control zawsze wysyła NL powierzchni `founder_autonomy` do jednego extractora
  `/conversation/founder/query` i nie poprawia jego semantyki regexami;
- usunięto lokalne klasyfikatory pytań o ticket, kolejkę, formularze, linki i
  sesję autonomii;
- brak LLM albo niepoprawny structured output zwraca HTTP 503,
  `llm_interpretation_unavailable`, `executed=false` i `query_dsl=null`;
- profil DOQL jest ponownie walidowany w Control, łącznie z odrzucaniem
  nieznanych pól na poziomie dokumentu i pojedynczego read modelu;
- NL sklasyfikowane przez LLM jako `operation` nie uruchamia lease ani innej
  mutacji. Wykonanie wymaga osobnego, zaakceptowanego DSL i grantu;
- LLM Gateway zwraca do audytu `response_id` obok modelu, providera i trybu
  structured output.

Testy po zmianie:

| Zakres | Wynik |
| --- | ---: |
| Founder CLI | 11/11 |
| Control: Founder + formularze + reconciliation | 114/114 |
| `@subactor/planfile-orchestration` | 33/33 |
| Runtime | 183/183 |

Po rebuildzie `hr-control` i `llm-gateway` były zdrowe, a hashe krytycznych
modułów host/kontener były identyczne. Control zakończył sesję z
`AUTONOMOUS_QUEUE_CONSUMERS_ENABLED=0`, `AUTONOMY_MUTATIONS_ENABLED=0` i
`PLESK_SYNC_APPLY=0`.

Dwa testy live przeszły przez ten sam extractor i różniły się wyłącznie
wygenerowanym DOQL:

- `Pokaż stan autonomii` → `organization_status`, odpowiedź query
  `gen-1785393356-3HjLforqfMtLhRNSU2jZ`;
- `Pokaż aktywne tickety automatyczne...` → `founder_work_status`, odpowiedź
  query `gen-1785393388-RhS7h9lwH2Pqbf8cDlJs`; odczyt: 0 gotowych, 2
  wykonywane, 18 oczekujących i 5 działań Foundera.

Response IDs są dowodem konkretnego wywołania, nie runtime dependency ani
substytutem receiptu wykonania.

## Kolejność realizacji na 30 lipca

### P0. Ustabilizować dzisiejszy baseline

- [ ] Zapisać osobno zakres zmian w `platform` i `core`; rozpoznać autorstwo
  zmian równoległych i nie mieszać ich w jednym commicie.
- [x] Usunąć heurystyczną klasyfikację NL w `subactor founder` i Control.
  Pytania `Pokaż stan autonomii` oraz `Pokaż aktywne tickety automatyczne` mają
  przechodzić przez ten sam LLM structured extractor, a różnić się wyłącznie
  wygenerowanym dokumentem DSL.
- [x] Ustawić dla NL tryb `require-llm`. Timeout, brak konfiguracji lub błędny
  structured output ma zwracać `llm_interpretation_unavailable`, bez fallbacku
  i bez wykonania domyślnej operacji.
- [x] Pozostawić jawne komendy API/CLI do diagnostyki bez LLM, ale nie nazywać
  ich interpretacją NL ani decyzją autonomiczną.
- [x] Ponownie uruchomić pełne zestawy testów Founder CLI, Control, runtime i
  planfile orchestration z aktualnego worktree.
- [x] Przebudować wyłącznie wymagane usługi i porównać hash krytycznych modułów
  host/kontener.
- [x] Po testach przywrócić i potwierdzić tryb bezpieczny:
  `AUTONOMOUS_QUEUE_CONSUMERS_ENABLED=0`, `AUTONOMY_MUTATIONS_ENABLED=0` oraz
  `PLESK_SYNC_APPLY=0`.
- [ ] Przygotować małe, tematyczne commity. Nie wykonywać zbiorczego commita
  całego współdzielonego worktree.

Warunek ukończenia: bieżąca funkcja zarządzania przez NL jest odtwarzalna po
czystym rebuildzie, testy są zielone, a lista wdrożonych commitów jest jawna.

### P0. Odblokować jeden kompletny ticket automatyczny

Pierwszym kandydatem jest `PLF-1943` dla `founder.subactor.com`. Ticket ma
gotową propozycję runtime twin, ale blokuje go
`runtime_twin_review_required:plesk-3953c8b62b726813b948`.

- [ ] Zmaterializować wymagane etapy review jako jawne subtickety albo exact URI:
  `human-baseline-review`, `connector-manifest-route-conformance` i
  `signed-baseline-attestation`.
- [ ] Każdemu review przypisać wykonawcę, input, spodziewany receipt i relację
  do `PLF-1943`. Nie używać ogólnego `continue` jako substytutu dowodu.
- [ ] Części maszynowe wykonać przez read-only Process Pack. Tylko rzeczywista
  decyzja człowieka może trafić do jednego pełnego formularza z historią.
- [ ] Po pozytywnym review ponownie policzyć readiness i sprawdzić, czy ticket
  przechodzi z `waiting_input` do `ready` bez ręcznej zmiany stanu.
- [ ] Wydać krótką lease wyłącznie dla jednego ticketu, uruchomić pojedynczy
  cykl, odebrać receipt i natychmiast cofnąć lease.

Warunek ukończenia: co najmniej jeden zwykły ticket przechodzi rzeczywistą
ścieżkę `waiting_input → ready → running → done` z niezależnym EQL read-backiem,
bez ręcznego oznaczenia sukcesu i bez rozszerzenia authority.

### P0. Naprawić kontrakt `founder.subactor.com`

Obserwowany stan nie spełnia kontraktu aplikacji. Root zwraca HTTP 200, ale
jest niewłaściwą statyczną treścią i nie ma Basic Auth. Trasy `/founder`,
`/founder/form` i `/founder/action` zwracają 404 bez challenge auth. Monitor
projektu słusznie utrzymuje projekt jako `application_route_not_ready`.

- [ ] Po review twin wskazać w manifeście osiągalny, uwierzytelniony upstream;
  nie zgadywać adresu i nie używać loopbacku jako produkcyjnego kontraktu.
- [ ] Wykonać dry-run publikacji i zachować `plan_hash` oraz pełny diff planu.
- [ ] Przed apply sprawdzić, czy plan obejmuje aplikację, reverse proxy, Basic
  Auth, publiczną trasę akcji i rollback.
- [ ] Apply wykonać dopiero po grancie związanym z dokładnym `plan_hash`.
- [ ] Wykonać niezależny read-back z zewnątrz hosta:
  - chroniony root lub panel zwraca 401 i `WWW-Authenticate` bez credentiali;
  - poprawne credentiale zwracają 200 dla panelu;
  - publiczna jednorazowa akcja/formularz ma wyłącznie zamierzony wyjątek od
    Basic Auth i nie ujawnia tokenu;
  - treść i routing odpowiadają źródłu projektu, nie placeholderowi;
  - monitor statusu przestaje raportować 404.
- [ ] Zamknąć lub oznaczyć jako zastąpione starsze tickety tylko po porównaniu
  ich oczekiwań z nowym receiptem. Zachować relację `supersedes`, zamiast usuwać
  historię.

Warunek ukończenia: projekt jest `converged`, a status wynika z HTTP, auth,
treści i route read-backu, nie tylko z udanego wywołania Plesk.

### P1. Pozostałe projekty domenowe

| Projekt | Aktualny blocker | Następne zadanie | Dowód ukończenia |
| --- | --- | --- | --- |
| `status-subactor-com` | `application_route_not_ready` (`PLF-1055`, `PLF-1056`) | Zbudować sitemap/runtime twin, wykonać dry-run trasy i porównać publiczny JSON z kontraktem | właściwa trasa, schema subset i read-back HTTP |
| `docs-stage-subactor-com` | `mutation_gate_disabled` (`PLF-886`, decyzja `PLF-1825`) | Founder opisuje decyzję w NL, LLM tworzy DSL; po akceptacji hasha osobny ticket apply | receipt publikacji, TLS i zgodność treści staging |
| `docs-subactor-com` | `mutation_gate_disabled` (`PLF-888`, `PLF-1826`) | Najpierw porównać fingerprint stage/production, potem dry-run i master gate | zgodny fingerprint, linki i read-back publikacji |
| `www-subactor-com` | `mutation_gate_disabled` (`PLF-898`, `PLF-1827`) | Odświeżyć plan względem aktualnego redirectu 301; usunąć nieaktualne kroki | oczekiwany redirect, TLS, treść celu i receipt |
| `autonomicznosc-pl` | `mutation_gate_disabled` (`PLF-1052`, `PLF-1934`) | Zastąpić ogólny formularz przepływem NL → LLM → DSL i aktualnym diffem | akceptacja związana z DSL hash i plan_hash oraz niezależny read-back |
| `contracts-subactor-com` | brak bieżącego blockera w projekcji | Wykonać tylko kontrolny post-deploy check i nie tworzyć nowego ticketu, jeśli kontrakt nadal jest zielony | HTTP/TLS/content receipt bez nowych zadań |
| `logo-subactor-com` | brak bieżącego blockera | Pozostawić w obserwacji; nie wdrażać ponownie bez driftu | zielony monitor bez mutacji |

Dla każdego projektu reconciler powinien przed utworzeniem nowego ticketu
porównać: cel, zasób, oczekiwania, snapshot/twin hash i aktywne relacje.
Duplikat pełny należy połączyć z ticketem kanonicznym, duplikat częściowy
rozbić na nadal aktualne oczekiwania, a przestarzały wariant zamknąć jako
`superseded` z odnośnikiem do następcy.

### P1. Brakujące capability platformy

- [ ] `PLF-682` / `PLF-1103`: zaimplementować rzeczywiste adaptery dostawy
  komunikacji; ludzka odpowiedź pozostaje jawnie `blueprint-only`.
- [ ] `PLF-683` / `PLF-1172`: dodać wyłącznie stabilne adaptery dashboardu,
  reconciliation i recruitment z idempotency oraz read-backiem.
- [ ] `PLF-684` / `PLF-1171`: oprzeć secret intake na one-time vault i evidence
  API. Żaden secret ani credential nie może wejść do URI, ticketu lub logu.
- [ ] Powiązać producenta `group-resolution:v1` z Control. Pakiet
  `@subactor/planfile-orchestration` ma mechanikę splitu, ale producent ticketów
  grupowych nie jest jeszcze podłączony do `server.mjs`.
- [ ] Przetestować deduplikację grup, `proceed_low_risk`, częściowe ukończenie
  i brak rekurencji formularz → formularz.
- [ ] Zaimplementować wykonywalny repair loop jako Process Pack:
  `problem fact → kandydaci → plan → AQL → command → EQL → receipt`.
  Sama diagnoza albo propozycja nie może zamykać ticketu naprawczego.
- [ ] Dodać usługę korelacji i higieny ticketów albo wydzielić ją jako jawny
  etap planfile orchestration. Powinna wykrywać exact/partial duplicates,
  stale expectations i relacje `supersedes`, ale nie kasować historii.

### P2. Wyodrębnić autonomię jako wersjonowany DSL

- [ ] Utworzyć paczkę `@subactor/autonomy-dsl` ze schematem źródłowym
  `subactor.autonomy-plan/v1`.
- [ ] Dodać obowiązkową kopertę źródła: surowe NL człowieka, content hash,
  aktor, timestamp, kanał, wersja promptu, model/provider i response ID.
- [ ] Zaimplementować jeden LLM structured extractor w trybie `require-llm`.
  Nie utrzymywać równoległego parsera semantycznego ani regexowego fallbacku.
- [ ] DSL ma opisywać fakty wejściowe, selektor zasobów, kroki URI, politykę
  AQL, oczekiwania EQL, retry, kompensację, split/join, human boundary i receipt.
- [ ] Runtime zawsze nadpisuje stan wyniku LLM na `proposed`, liczy stabilne ID
  i canonical hash oraz odrzuca nieznane pola i referencje.
- [ ] Dodać semantic diff `NL/DSL poprzedni → DSL proponowany` i akceptację
  człowieka związaną z konkretnym hashem. Sama odpowiedź modelu nie jest
  authority.
- [ ] Kompilować DSL do istniejącego Process Envelope v2. DSL nie może być
  drugim runtime ani omijać AQL/OQL/URI/EQL.
- [ ] Udostępnić read-only API: `validate`, `compile`, `simulate` i `explain`.
  `execute` pozostaje po stronie istniejącego kontrolera i authority gates.
- [ ] LLM jest jedyną warstwą semantycznej interpretacji NL i może proponować
  tylko typowany dokument DSL. Nie może dostarczać
  dowolnego URL-a, polecenia shell, credentiala ani samodzielnie nadawać sobie
  authority.
- [ ] Jako pierwszy slice przenieść z hardcodowanego
  `project-remediation.mjs` reakcję na
  `founder-subactor-com/application_route_not_ready`.
- [ ] Uruchomić shadow comparison starej i nowej strategii, potem ograniczony
  canary dla jednego projektu. Usunięcie starej ścieżki dopiero po zgodności
  planów i receiptów.

Docelowa zależność pozostaje jednokierunkowa:

```text
human NL → LLM require-llm → proposed autonomy DSL → human/policy hash acceptance
             → API → accepted DSL → Process Envelope v2
             → AQL → OQL → exact URI connector → EQL → receipt
```

## Decyzje, których system nie powinien podejmować za Foundera

Te zadania należy pokazać z pełnym kontekstem i historią, ale nie wolno ich
automatycznie zatwierdzać:

- `PLF-653` — warunki i uruchomienie publikacji stażu;
- `PLF-849` — warunki pierwszego płatnego pilota;
- `PLF-1052` — zgoda na skutek publikacji `autonomicznosc-pl`;
- `PLF-1322` — Digital Twin human checkpoint;
- `PLF-1825` — publikacja `docs-stage-subactor-com`.

Formularz jest tylko interfejsem zebrania wypowiedzi człowieka. Musi zachować
NL, a następnie pokazać DSL wygenerowany przez LLM i semantic diff przed
akceptacją hasha. Powinien odpowiadać na cztery pytania: co już działa, co jest
zablokowane, jaki dokładnie skutek ma decyzja i jaki dowód powstanie po
wykonaniu. Ogólne `Kontynuuj` nie jest poprawną odpowiedzią na brak
specyficznego expectation.

## Macierz testów przed uznaniem wdrożenia

### Testy kodu

```bash
cd /home/tom/github/subactor/platform/packages/founder-cli
npm test

cd /home/tom/github/subactor/core
node --test services/control/tests/founder-chat-query.test.mjs \
  services/control/tests/founder-form-catalog.test.mjs \
  services/control/tests/founder-form-routes.test.mjs \
  services/control/tests/project-reconciliation.test.mjs \
  services/control/tests/project-ticket-portfolio-reconciliation.test.mjs

cd /home/tom/github/subactor/platform/packages/planfile-orchestration
npm test

cd /home/tom/github/subactor/runtime
npm test
```

### Test kontrolera i projekcji NL

```bash
cd /home/tom/github/subactor/platform
./bin/subactor founder \
  "Pokaż aktywne tickety automatyczne: gotowe, wykonywane, oczekujące oraz pięć działań Foundera." \
  --json
make control-execute-once
```

Po cyklu sprawdzić, że liczba gotowych/wykonanych wynika z receiptów, nie z
samego statusu lifecycle, oraz że nie powstały kolejne formularze o
formularzach ani seria identycznych emaili.

### Testy publiczne po deployu

```bash
curl -sS -o /dev/null -D - https://founder.subactor.com/
curl -sS -o /dev/null -D - https://founder.subactor.com/founder
curl -sS -o /dev/null -D - https://status.subactor.com/
```

Dla chronionej trasy oczekiwany jest jawny challenge Basic Auth. Test z
credentialami należy wykonać przez bezpieczny credential handle; nie wpisywać
sekretu do dokumentu, historii shell ani URL-a.

### Walidacja dokumentów zarządzanych

```bash
cd /home/tom/github/subactor/platform
npm run artifacts:build
npm run artifacts:check
```

## Reguły zatrzymania

- Nie włączać globalnych mutacji tylko po to, aby kolejka wyglądała na
  aktywną. Używać krótkiej lease ograniczonej do konkretnego planu/ticketu.
- Nie interpretować NL bez LLM i nie wykonywać semantycznego fallbacku.
- Nie traktować wyniku LLM jako zaakceptowanej decyzji; wymagany jest hash
  akceptacji człowieka albo wcześniejszego policy grantu.
- Nie odpowiadać automatycznie na human boundary i nie traktować przypomnienia
  jako nadania authority.
- Nie zamykać ticketu po samym HTTP 2xx komendy. Wymagany jest niezależny
  read-back i spełnione expectation.
- Nie tworzyć kolejnego formularza dla ticketu, który sam jest formularzem.
- Nie wysyłać ponownie identycznej notyfikacji bez zmiany stanu, nowego
  dowodu albo upływu zdefiniowanego reminder window.
- Nie oznaczać starego ticketu jako duplikatu wyłącznie na podstawie nazwy;
  porównywać zasób, cel, oczekiwania i snapshot hash.
- Po każdym teście mutacji cofnąć lease i potwierdzić bezpieczne flagi.

## Definicja zakończenia jutrzejszej sesji

Sesję można uznać za zakończoną, gdy:

- baseline kodu jest odseparowany i wszystkie wskazane testy są zielone;
- jeden nie-testowy ticket został wykonany przez pełny kontrolowany lifecycle
  albo ma udokumentowany, konkretny blocker bez generowania duplikatów;
- `founder.subactor.com` spełnia kontrakt auth/routingu albo ma jeden aktualny
  plan związany z plan_hash i przypisanym review;
- każdy z sześciu zablokowanych projektów ma dokładnie jeden aktualny następny
  krok i właściciela;
- kontroler nie produkuje rekurencyjnych formularzy ani powtarzanych emaili;
- stan końcowy bramek jest bezpieczny i zapisany w raporcie z receiptami;
- wyniki, commity, wdrożone obrazy i otwarte blockery zostały dopisane jako
  następna wersja tego dokumentu, bez nadpisywania historii wersji 3.
