---
{
  "schema": "subactor.doc/v1",
  "id": "docs.architecture.autonomy-evolution-loop",
  "version": 5,
  "status": "current",
  "updated": "2026-07-23"
}
---

# Autonomy Evolution Loop

## Cel

Autonomia ma zwiększać ilość poprawnie zakończonej pracy na jednostkę czasu,
kosztu, ryzyka i uwagi człowieka. Liczba cykli kontrolera, wywołań LLM albo
utworzonych ticketów nie jest miarą sukcesu.

System może proponować i testować własne ulepszenia, lecz nie może:

- sam rozszerzyć AQL lub root authority;
- zmienić Constitution bez Foundera albo wymaganego quorum;
- promować zmiany na podstawie własnego receipt;
- ominąć canary, niezależnego EQL albo rollbacku;
- uznać częściowego snapshotu za pełną sytuację;
- tworzyć wielu kandydatów dla tego samego fingerprintu.

## Pętla

```text
Observe -> Aggregate -> Diagnose -> Select
   ^                                |
   |                                v
Learn <- Measure <- Promote <- Experiment
                  \- Rollback/Retire
```

### Observe

SODL zapisuje zdarzenia, receipts, koszty i odpowiedzialność. Wymagane są
stabilne correlation ID, fingerprint, czas monotoniczny i jawna świeżość.

### Aggregate

DOQL buduje sytuację oraz trendy. Każde źródło deklaruje expected scope,
observed count, freshness, snapshot hash i completeness receipt.

### Diagnose

DQL sprawdza bezpieczeństwo, liveness, progress, currency, uniqueness,
independence, recoverability i coverage. Finding nie jest jeszcze przyczyną
ani uprawnieniem do naprawy.

### Select

Jeden ProblemCase grupuje wspólną przyczynę. Improvement Candidate ma
hipotezę, baseline, przewidywaną wartość, koszt, ryzyko, confidence,
odwracalność, właściciela i limit zasobów.

### Experiment

Strategy DSL wybiera dozwoloną reakcję, AQL ogranicza principal, OQL opisuje
operacje, URI Process wykonuje przypięte kroki, a EQL określa niezależny
rezultat. Eksperyment zaczyna się od symulacji i izolowanego canary.

### Promote

Poziomy dojrzałości capability:

1. `observe`;
2. `recommend`;
3. `simulate`;
4. `canary`;
5. `bounded_execute`;
6. `autonomous`;
7. `self_improving`.

Promocja zwiększa tylko wcześniej zatwierdzony zakres. Awaria cofa capability
o poziom albo uruchamia rollback.

### Measure and Learn

Raport before/after porównuje lead time, throughput, retry, rollback, rework,
koszt, liczbę ręcznych reakcji oraz jakość EQL. Wynik aktualizuje Knowledge,
profil DQL i katalog strategii albo prowadzi do wycofania eksperymentu.

## Budżety

Każdy eksperyment ma:

- limit czasu i prób;
- limit tokenów oraz kosztu API;
- limit ticketów i równoległości;
- dozwolony scope zasobów;
- limit mutacji;
- maksymalny blast radius;
- automatyczny stop;
- preimage i rollback;
- termin przeglądu wiedzy.

## Audyt live 2026-07-30

System ma prawie wszystkie elementy pętli, ale nie ma jeszcze jednego kontraktu,
który łączy wynik diagnostyki z wykonawcą napraw. Dlatego człowiek nadal wykonuje
pracę integratora: rozpoznaje finding, wybiera repozytorium, formułuje zmianę,
uruchamia test rzeczywisty i ocenia, czy wynik można promować.

| Warstwa | Co działa | Gdzie pętla się urywa |
|---|---|---|
| Monitor | Control, kontrolery, receipts, live probes i timer `autonom` | `autonom` zapisuje lokalne propozycje z `acts:false`; błąd timera nie staje się automatycznie ProblemCase |
| Analyze | DOQL/DQL i `/api/autonomy/control` rozróżniają human boundary, intentional gate, lifecycle stall i structural gap | klasyfikacja nie zawiera jeszcze wykonywalnego kontraktu naprawy |
| Plan | katalog remediacji, Strategy DSL, dry-run i plan hash | planner jest symulacyjny, a `autonomy.repair.canary-pilot` pozostaje w `shadow` |
| Execute | URI Process, kolejki botów oraz hostowy `coding-agent` | stare tickety `developer_implementation_required` nie są kompilowane do `subactor.coding-agent-task/v1` |
| Verify | AQL/OQL/URI/EQL, testy repozytorium i receipts | wykonawca kodu wystawia własne EQL; brakuje niezależnego runtime exercise i walidatora promocji |
| Learn | Knowledge jest append-only, artefakty mają hash i wersję | wynik naprawy nie aktualizuje automatycznie twina, katalogu strategii i poziomu capability |

Live Control raportował sześć luk strukturalnych: trzy źródłowe tickety
`PLF-682/683/684` i trzy tickety implementacyjne `PLF-1103/1171/1172`.
Wszystkie wskazują dokładne URI blueprintów, których connector runtime nie
serwuje. System zna więc problem i zamierzonego wykonawcę, ale nie potrafi
przekształcić starszego manifestu remediacji w kontrakt przyjmowany przez
hostowego konsumenta.

Sam konsument jest operacyjny. Timer `subactor-coding-agent.timer` odpytuje
Planfile co minutę, a `PLF-2186` przeszedł pełną ścieżkę: zwalidowany Process
Envelope, izolowany worktree z `origin/main`, `codex exec` w sandboxie, testy
repozytorium oraz completion receipt. To dowodzi, że brakującą warstwą nie jest
„uruchom LLM”, lecz deterministyczny **compiler diagnoza -> zadanie naprawcze**.

Audyt wykrył też dryf w samym obserwatorze. Sonda postawy była przypięta do
historycznego `PLF-2051`, mimo że aktywną, zweryfikowaną decyzją było
`PLF-2295 = production_apply`. Po zmianie sonda wybiera najnowszy ticket z
etykietą `autonomy-decision:v1`, zgodnie z twinem Control. Cykl zmienił wynik
z trzech naruszeń na jedno rzeczywiste naruszenie pinów komponentów.

## Docelowy compiler ewolucji

```text
Finding + snapshot hash
  -> deduplikowany ProblemCase
  -> klasyfikacja: known repair | code change | twin | connector | process pack
  -> Improvement Candidate + capability snapshot hash
  -> todo2code Intent Evidence / code-change plan
  -> Process Envelope AQL + OQL + exact URI + EQL
  -> izolacja -> dry-run -> canary -> runtime exercise
  -> niezależny Validator
  -> promote albo rollback
  -> nowa wersja Knowledge, twina i katalogu strategii
```

`todo2code` pasuje do etapów ugruntowania i planowania. Łączy task, Git, AST,
TODO i dokumentację w graf dowodów, wykrywa Intent vs Reality oraz tworzy
hash-bound code-change plan. Nie powinien przejmować AQL, uruchamiać efektów ani
sam promować zmiany. Jego wynik jest wejściem do Process Envelope, a nie zgodą.

### Typy kandydatów

1. **Known repair** — istniejący, podpisany Process Pack; bez LLM, jeśli
   fingerprint i preconditions pasują dokładnie.
2. **Code change** — plan `todo2code`, allowlista repozytoriów, izolowany
   worktree, testy kodu i obowiązkowy runtime exercise.
3. **Digital Twin** — discovery tworzy tylko kandydata; conformance, shadow,
   canary i podpis promocji tworzą nową wersję twina.
4. **Connector** — kontrakt tras, scaffold, testy, build obrazu, doctor,
   izolowany registry, canary, signed release i rollback do last-known-good.
5. **Process Pack** — luka procesu staje się kandydatem DSL; schema i artifact
   registry, symulacja, canary, niezależny EQL i dopiero aktywacja.

Automatyczna ścieżka nie może zwiększać własnej authority. Nowa trasa, twin lub
pack zaczyna co najwyżej na poziomie `observe` albo `simulate`. Przejście do
`bounded_execute` jest dozwolone tylko w istniejącym profilu ryzyka Foundera;
nowy scope, sekret, nieodwracalny efekt lub zmiana Constitution wymaga osobnej
decyzji człowieka związanej z dokładnym hashem kandydata.

## Priorytet wdrożenia

1. Emitować wyniki `autonom` jako typowane, deduplikowane ProblemCase w
   Planfile; dodać `OnFailure`/outbox, aby czerwony timer nie kończył się tylko
   statusem systemd.
2. Dodać compiler `developer_implementation_required ->
   subactor.coding-agent-task/v1`. Musi wybrać repozytorium z katalogu, zbudować
   ugruntowany prompt i zachować jawne flagi commit/push; nie może wykonywać
   dowolnego tekstu z historycznego ticketu.
3. Oddzielić wykonawcę od walidatora. Zielone testy kodu nie wystarczają:
   validator uruchamia nowy connector/twin/pack, wykonuje doctor i jeden
   prawdziwy proces bez skutku albo na canary oraz sprawdza EQL.
4. Dostarczyć trzy lifecycle packi: `twin.develop-promote`,
   `connector.develop-release` i `process-pack.develop-promote`, każdy z
   last-known-good oraz rollbackiem.
5. Po serii zielonych canary promować `autonomy.repair.canary-pilot` z `shadow`
   do `bounded_execute` wyłącznie dla fingerprintów i budżetów objętych decyzją
   Foundera.
6. Po każdej promocji zapisać nową, append-only wersję Knowledge oraz zmierzyć:
   czas do zweryfikowanej naprawy, odsetek napraw automatycznych, rollback rate,
   wiek twina, liczbę luk strukturalnych i uwagę człowieka na poprawny rezultat.

## Pierwszy backlog

1. Rozszerzyć bazowy `controller.progress` o SLA obowiązków, retry i
   eskalacje.
2. Zebrać co najmniej dwanaście cykli podtelemetrii `problem_reactions`, w tym
   co najmniej jeden rzeczywisty `ticket_creation`.
3. Finding Outbox i deduplikowany ProblemCase.
4. Receipts kompletności źródeł DOQL.
5. Migracja aktywnych ticketów do Constitution binding.
6. Analiza decyzji `notify`.
7. Zewnętrzny Sentinel.
8. Rozdzielenie runtime i principals Doctor, Repair oraz Validator.
9. Router modeli oparty o koszt, confidence i wpływ decyzji.

## Dowód początkowy

Schedule-aware liveness, izolacja instancji, progress oraz obsługa aktywnego
cyklu przeszły 14/14 testów i live read-back. Przy okresie 300000 ms oraz
jitterze 30000 ms próg wynosi 630000 ms.

Pierwszy cykl najnowszej instancji został zapisany jako `startup`. Miał zero
wykonywalnych ticketów i 33 formalne powiadomienia, ale utworzył jeden
deduplikowany ticket reakcji, dlatego progress wyniósł dokładnie jedną
jednostkę. Wcześniejszy live shadow podczas startu zwrócił `cycle_running` z
jawnym `cycle_id`, bez naruszeń i bez mutacji zewnętrznych.

To rozdziela trzy wcześniej mieszane sytuacje: poprawne oczekiwanie na start,
trwającą pracę i rzeczywisty zastój. Zaobserwowany czas startowych cykli był
zmienny (od 2302 ms do około 131 s), dlatego następny krok to telemetria
etapów, a nie arbitralne skrócenie timeoutu. Profil nadal nie mierzy wieku
najstarszego obowiązku ani skuteczności późniejszej reakcji człowieka lub
bota.

Telemetria etapów została wdrożona jako zamknięty kontrakt siedmiu kroków.
Pierwszy read-back przypisał 2523 z 2623 ms do `problem_reactions`. Etap
utworzył jeden ticket dla nowego fingerprintu; ostatnie tickety reakcji miały
różne fingerprinty. Wniosek nie brzmi więc „usuń duplikaty”, lecz: rozdziel
czas wyboru kandydata, kompilacji manifestu, zapisu Planfile i audytu oraz
porównaj kilka cykli przed optymalizacją.

Podetapy `problem_reactions` są już osobnym kontraktem. Pierwszy live cycle po
wdrożeniu nie miał kandydata: selection zakończył się poprawnie, a manifest,
Planfile create i audit zostały jawnie pominięte. Dzięki temu cykl bez pracy
nie zanieczyszcza percentyli wykonania i nie udaje postępu. Następna decyzja
optymalizacyjna wymaga naturalnego lub testowego, izolowanego przypadku z
rzeczywistym kandydatem; produkcyjnego błędu nie należy wywoływać wyłącznie w
celu zebrania metryki.
