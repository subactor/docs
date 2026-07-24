---
{
  "schema": "subactor.doc/v1",
  "id": "docs.architecture.autonomy-evolution-loop",
  "version": 4,
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
