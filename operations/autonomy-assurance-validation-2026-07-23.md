---
{
  "schema": "subactor.doc/v1",
  "id": "docs.operations.autonomy-assurance-validation-2026-07-23",
  "version": 7,
  "status": "current",
  "updated": "2026-07-23"
}
---

# Walidacja Constitution, Assurance i Controller

## Cel

Runbook sprawdza, czy Subactor:

- rozpoznaje aktualny, wykonywalny i nieaktualny ticket;
- zatrzymuje nieuprawnioną lub niekompletną operację;
- wykrywa anomalie kontrolera i architektury;
- przygotowuje tylko przypiętą, idempotentną naprawę;
- wymaga niezależnej walidacji i rollbacku;
- nie myli braku postępu z awarią ani częściowego snapshotu z pełną sytuacją.

Procedura jest domyślnie read-only. Wywołania z `--apply`, publikacja evidence
do Control i produkcyjne URI Process wymagają osobnej decyzji oraz AQL.

## Model podlegający testom

```text
Constitution
  |
  +-- validates authority, delegation and version binding
  |
Assurance Supervisor
  +-- DOQL: aggregate situation
  +-- DQL: evaluate invariants
  +-- Doctor: diagnose without mutation
  +-- Repair: pinned candidate in canary
  +-- Validator: independent EQL/read-back
  |
Controller
  +-- readiness + currency
  +-- claim + URI execution
  +-- receipts + completion
  +-- reminders + lifecycle reconciliation
```

## Poziom 1 — walidacja artefaktów

```bash
cd /home/tom/github/subactor/platform
npm run artifacts:build
npm run artifacts:check
```

Wynik zaliczony:

- brak findings Artifact Registry;
- Constitution, profile DQL/DOQL, AQL/OQL/EQL i URI Process mają immutable
  revision URL;
- zmieniony dokument z `version_source=declared` ma podniesioną wersję.

## Poziom 2 — DQL i DOQL

```bash
cd /home/tom/github/subactor
node platform/scripts/run-diagnostic-profile.mjs
node platform/scripts/run-doql-situation.mjs
```

Wynik zaliczony:

- DQL ma `ok=true`, wszystkie wymagane wejścia oraz zero findings;
- DOQL ma `read_only=true` i `automatic_mutation_allowed=false`;
- raport zawiera hash profilu i snapshotu;
- zakres wyniku jest opisany jako obserwowany zbiór, nie cała organizacja bez
  dowodu kompletności.

Aktualny profil architektury sprawdza tylko trzy inwarianty i korzysta ze
statycznego odczytu tras. Zielony wynik nie zastępuje live route registry.

## Poziom 3 — Constitution i lifecycle

```bash
cd /home/tom/github/subactor/platform
node --test \
  test/organization-constitution.test.mjs \
  test/lifecycle-type-registry.test.mjs \
  test/delegation-remediation-invariant.test.mjs
```

Wynik zaliczony:

- aktywna Constitution jest poprawna i hash-bound;
- nieznany typ encji powoduje błąd inventory;
- ticket naprawczy nie może utworzyć kolejnego ticketu naprawczego;
- istniejąca rekurencja daje finding, a terminalna historia nie jest
  traktowana jako aktywne naruszenie.

## Poziom 4 — anomalie i stress bez efektów zewnętrznych

```bash
cd /home/tom/github/subactor/autonomy-lab
npm run check
npm run stress
npm run canary:stress
```

Minimalna macierz:

| Anomalia | Oczekiwany wynik |
|---|---|
| brak exact URI | brak promocji do ready |
| trasa pojawia się po błędzie | jeden idempotentny recheck |
| burst tego samego błędu | jeden cykl po deduplikacji |
| częściowe wykonanie | zachowane receipts ukończonych kroków |
| URI zaproponowane przez LLM | odmowa |
| brak evidence propozycji LLM | odmowa |
| błąd zapisu Repair | brak mutacji zewnętrznej |
| czerwony Validator | rollback |
| zmiana `candidate_hash` | odmowa |
| duplikat wykonania | jeden efekt i duplicate receipt |

Wynik zaliczony wymaga `external_mutations=0` i zera nieoczekiwanych failures.

## Poziom 5 — read-only stan live

Token należy przekazać przez środowisko procesu. Nie wolno umieszczać go w URL,
logu, tickecie ani raporcie.

```bash
cd /home/tom/github/subactor/autonomy-lab
SUBACTOR_ADMIN_TOKEN='wartość-z-bezpiecznego-źródła' npm run e2e:live
```

Następnie należy wykonać uwierzytelnione:

```text
POST /api/tickets/lifecycle/reconcile
{"apply": false}
```

Sprawdzane pola:

- `controller_observed_at` i skonfigurowany `periodic_interval_ms`;
- `controller_instance_id`, `cycle_id`, `cycle_started_at`,
  `cycle_completed_at` i czas trwania;
- `controller_stage_telemetry`, kompletność siedmiu etapów, ich kolejność,
  sumę i najwolniejszy etap;
- `summary.executable`, `notifications`, `blocked` i `ignored`;
- `currency_current`, `currency_uncertain`, `currency_obsolete`;
- `constitution_current`, `constitution_legacy_unbound`,
  `constitution_stale`;
- liczba rechecków, promocji i odświeżonych diagnoz;
- brak mutacji zewnętrznych.

Observer wylicza próg jako
`max(120000, 2 * periodic_interval_ms + jitter_budget_ms)`. Brak lub błędny
harmonogram daje `controller_schedule_invalid`, obserwacja z przyszłości poza
budżetem daje `controller_clock_skew`, a okres przekraczający dobę daje
`daily_liveness_not_met`.

Dozwolone stany podczas startu:

- `startup_pending` — nie minął jeszcze jawny grace i cykl nie wystartował;
- `cycle_running` — cykl ma ID i czas startu, ale nie wystawił jeszcze
  completion eventu;
- `ready` — istnieje świeży, ukończony cykl bieżącej instancji.

`cycle_running` jest zielony tylko do budżetu stale. Po jego przekroczeniu
observer wystawia `controller_cycle_timeout`. `running=true` bez
`cycle_started_at` daje `controller_cycle_start_missing`.

Jeżeli scheduler reklamuje
`cycle_telemetry_schema=subactor.controller-cycle-telemetry/v1`, ukończony
cykl musi zawierać wszystkie etapy w ustalonej kolejności. Brak danych daje
`controller_stage_telemetry_unavailable`, zmiana cyklu
`controller_stage_telemetry_cycle_mismatch`, a niepełny wynik
`controller_stage_telemetry_incomplete`. Podczas `cycle_running` stan
telemetrii jest `collecting`, nie `unavailable`.

Jeżeli scheduler reklamuje
`problem_reaction_telemetry_schema=subactor.problem-reaction-telemetry/v1`,
observer wymaga czterech podetapów w ustalonej kolejności. Dla braku kandydata
dozwolony jest wyłącznie wynik:

```text
candidate_selection = completed
manifest_compilation = skipped:no_candidate
ticket_creation = skipped:no_candidate
audit_receipt = skipped:no_candidate
candidate_count = 0
tickets_created = 0
```

Niespójność liczby kandydatów i ticketów, pominięcie pracy przy istniejącym
kandydacie, błędny `cycle_id` albo niewłaściwa suma czasów są violations.

Progress jest liczony wyłącznie z eventów bieżącego `controller_instance_id`.
Jednostkami postępu są wykonane tickety, naprawy lifecycle, ukończone
formularze, utworzone tickety reakcji, promocje readiness i nowe relacje
currency. Sama liczba powiadomień nie jest postępem. Dwa kolejne cykle z
`executable>0` i zerem jednostek postępu dają
`controller_progress_stalled`; `executable=0` z formalnymi powiadomieniami
daje `waiting_external`.

## Poziom 6 — shadow i produkcyjny canary

Shadow:

- zapisuje findings i RepairCandidate;
- nie tworzy grantów;
- nie wykonuje produkcyjnych URI;
- porównuje decyzję obecnego kontrolera z planowaną polityką `enforce`.

Produkcyjny canary może dotyczyć wyłącznie odwracalnego, ograniczonego zasobu.
Musi mieć:

- jeden ticket i jeden `candidate_hash`;
- jawny plan hash i minimalny AQL;
- preimage oraz rollback URI;
- limit czasu i prób;
- niezależnego Validatora;
- EQL oparty o publiczny lub niezależny read-back;
- automatyczne zatrzymanie po czerwonym wyniku.

## Warunki przejścia `observe` → `enforce`

Przejście jest dozwolone dopiero, gdy jednocześnie:

- aktywne `constitution_legacy_unbound == 0`;
- `constitution_stale == 0`;
- dwa pełne okna kontrolera nie mają niewyjaśnionego zastoju;
- live observer używa progu zależnego od harmonogramu;
- źródła DOQL mają receipts kompletności i świeżości;
- Finding Outbox jest idempotentny;
- Doctor, Repair i Validator mają rozdzielone principals oraz AQL;
- stress, canary, rollback i niezależny read-back są zielone;
- zewnętrzny Sentinel wykrywa zatrzymanie Control.

## Warunki natychmiastowego przerwania

- nieznana wersja kontraktu albo Constitution;
- brak snapshotu wymaganego przez DQL/DOQL;
- LLM proponuje URI, authority, sekret lub shell;
- Repair i Validator mają ten sam principal lub credential;
- brak preimage/rollback dla mutacji;
- EQL nie wskazuje konkretnego `verified_by`;
- liczba aktywnych RepairCandidate dla fingerprintu przekracza jeden;
- green report nie ma dowodu kompletności źródeł;
- observer i scheduler używają sprzecznych zegarów.

## Wynik bazowy z 2026-07-23

- DQL Control: 3/3.
- DOQL output integrity: 2/2.
- Portfolio manifestów: 8/8 domen, bez dowodu kompletności Organization Core.
- Anomaly suite: 17/17.
- Controller stress: 1700/1700.
- Repair canary stress: 600/600.
- Constitution/lifecycle/remediation: 7/7.
- Live lifecycle: 32 aktywne, 32 current, 32 legacy unbound, 31 notify,
  1 ignore, 0 execute.
- Live observer po poprawce: wiek 218164 ms, okres 300000 ms, jitter 30000 ms,
  próg stale 630000 ms, `ok=true`, zero violations i zero mutacji
  zewnętrznych.
- Schedule-aware liveness, izolacja instancji, aktywny cykl i progress:
  14/14.
- Celowane testy Core kontrolera: 23/23.
- Live po restarcie: odrębny `controller_instance_id`, pierwszy completion z
  triggerem `startup`, 54 scanned, 33 considered, 0 executable, 0 blocked,
  33 notifications, czas 2302 ms oraz jeden utworzony ticket reakcji.
  Progress: `productive`, jedna jednostka postępu, zero mutacji zewnętrznych.
- Live shadow podczas następnego startu: `cycle_running`, jawny `cycle_id`,
  brak violations i zero mutacji zewnętrznych.
- Stage telemetry live: 2627 ms całego cyklu, 2623 ms etapów;
  `problem_reactions=2523 ms`, `snapshot_load=62 ms`,
  `queue_execution=22 ms`, pozostałe etapy łącznie 16 ms. Schemat kompletny,
  zgodny z `cycle_id`, zero violations i zero mutacji zewnętrznych.
- Po rozszerzeniu telemetrii: Core 27/27, Autonomy Lab 16/16, stress
  1700/1700.
- Pełny Core: 642 zaliczone, 7 pominiętych, 0 błędów.
- Po dodaniu podetapów reakcji: celowane Core 29/29, Autonomy Lab 18/18,
  stress 1700/1700, pełny Core 644 zaliczone, 7 pominiętych, 0 błędów.
- Live bez nowego kandydata: 88 ms etapów całego cyklu,
  `candidate_count=0`, `tickets_created=0`, trzy jawne
  `skipped:no_candidate`, `waiting_external`, zero violations i zero mutacji
  zewnętrznych.
- Ręczna rekoncyliacja o `2026-07-23T20:47:45.289Z`: 8 projektów, z czego
  `logo-subactor-com` jest `converged`, a 7 ma nadal obserwowalne blockery.
  Kontroler po rekoncyliacji: 55 ticketów, 35 otwartych, 20 zakończonych,
  28 `waiting_input`, 5 `ready`, 2 `running`, 0 niespójności lifecycle.
- Stan domen projektowych: `autonomicznosc-pl`, `docs-stage-subactor-com`,
  `docs-subactor-com` i `www-subactor-com` czekają na kontrolowaną bramę
  publikacji; `contracts-subactor-com` czeka kolejno na DNS, TLS i aplikację;
  `founder-subactor-com` ma poprawne DNS/TLS, lecz `/founder/action` nadal
  zwraca 404; `status-subactor-com` ma niezgodny artefakt aplikacji.
- Środowisko produkcyjne pozostaje fail-closed: master gate, auto-apply i
  wszystkie szczegółowe bramy Plesk są wyłączone. Istniejące zgody dla
  dokładnych `plan_hash` nie zostały wykonane i nie zostały rozszerzone.
- Skorygowano niepełne podsumowania dry-run w `PLF-913` i `PLF-914` na
  podstawie ich terminalnych ticketów wykonania, bez ponownego URI Process i
  bez `apply=true`: po 134 pliki oraz odpowiednio 2 200 242 i 2 200 235
  bajtów. `plan_hash` pozostał bez zmian.
- Zielony receipt publikacji jest od tej wersji użyteczny tylko, gdy ma
  poprawny `plan_hash`, całkowite `file_count`, skończone `byte_count` oraz
  jawny boolean `changed`. Niepełny legacy receipt jest automatycznie
  odświeżany przez bezpieczny dry-run zamiast bezwarunkowo pomijać preflight.
- Test regresyjny receipt i formularzy: 35/35. Pełny Core po zmianie:
  646 zaliczonych, 7 pominiętych, 0 błędów.
- Odpowiedzialność po zatwierdzeniu dokładnego `plan_hash` jest kierowana do
  `administrator-bot`, a Founder pozostaje wyłącznie odbiorcą eskalacji.
  `mutation_gate_disabled` z jawnym właścicielem `bot/machine/system` nie może
  już tworzyć kolejnego formularza Foundera.
- Fallback formularza lifecycle nie jest generowany dla ticketu z
  `blocked-by:*`; użytkownik odpowiada na rzeczywisty ticket zależności zamiast
  na równoległą projekcję przyszłej pracy.
- Kontroler wygasił jako `canceled/superseded` dwa osierocone tickety
  formularzy `PLF-1178` i `PLF-1186`. Historia i relacja `source:*` zostały
  zachowane; źródłowe tickety nie zostały uznane za wykonane.
- Ręczny cykl po wdrożeniu: 53 scanned, 32 considered, 1 executable,
  31 notifications, 0 blocked. Kontroler wykonał `PLF-1190` przez cztery
  konkretne URI Process (`read`, `record`, `classify`, `audit`) i zamknął go
  z kompletnym EQL receipt.
- Aktualny Planfile po cyklu: 52 tickety, 32 aktywne, w tym 26
  `waiting_input`, 5 `ready` i 1 ciągły kontroler `running`. Dwa formularze są
  anulowane, 17 ticketów jest `done`, jeden historyczny ticket diagnostyczny
  pozostaje `failed`.
- Pełny Core po zmianach odpowiedzialności i currency formularzy:
  649 zaliczonych, 7 pominiętych, 0 błędów. Testy celowane zmienionego
  przepływu: 63/63.

## Autonomy Evolution Loop

Doskonalenie systemu jest osobnym lifecycle, a nie nieograniczonym prawem
Control do modyfikowania siebie:

```text
SODL observations and receipts
              |
              v
DOQL situation, trends and resource usage
              |
              v
DQL finding or improvement opportunity
              |
              v
one ProblemCase / one Improvement Candidate
              |
              v
value-cost-risk selection
              |
              v
Strategy DSL + AQL + OQL + pinned URI Process
              |
              v
simulation -> isolated canary -> independent EQL
              |
       +------+------+
       |             |
       v             v
bounded promote   rollback / retire
       |
       v
before/after receipt -> Knowledge -> new invariant
```

### Procesy

1. `autonomy.improvement.observe/v1` mierzy lead time, czas w stanie, liczbę
   cykli bez postępu, retry, rollback, rework, koszt LLM/API, uwagę człowieka
   oraz udział efektów potwierdzonych niezależnym EQL.
2. `autonomy.improvement.detect/v1` wykrywa wspólne przyczyny, błędne
   `waiting_input`, brak exact URI, słabe EQL, nieskuteczne przypomnienia i
   zielone raporty bez kompletności źródeł.
3. `autonomy.improvement.select/v1` wybiera ograniczoną liczbę eksperymentów
   według wartości, częstotliwości, confidence, odwracalności, kosztu, ryzyka
   i kosztu koordynacji.
4. `autonomy.improvement.experiment/v1` przypina hipotezę, baseline,
   `candidate_hash`, `plan_hash`, budżet, canary, EQL, rollback oraz warunek
   zatrzymania.
5. `autonomy.improvement.promote/v1` przechodzi stopniowo przez `observe`,
   `recommend`, `simulate`, `canary`, `bounded_execute`, `autonomous` i
   `self_improving`.
6. `autonomy.improvement.review/v1` porównuje before/after i promuje, wycofuje
   albo usuwa zmianę oraz aktualizuje Knowledge i inwarianty.

### Priorytet ograniczonych zasobów

```text
priority =
  expected_value
  * occurrence_frequency
  * evidence_confidence
  * reversibility
  / (implementation_cost + operational_risk + coordination_cost)
```

System powinien preferować deterministyczne reguły dla lifecycle, authority,
hashy, exact URI, idempotency i completion. LLM jest uzasadniony dla
niejednoznacznej diagnozy, interpretacji NL, generowania ograniczonych
wariantów i oceny komunikacji. Droższy model jest fallbackiem po błędzie
schematu, niskim confidence albo przy wysokim koszcie błędnej decyzji.

### Najbliższe analizy

- lejek `created -> assigned -> ready -> running -> verified -> done`;
- przyczyny 31 decyzji `notify` i zera `execute`;
- exact URI blokujące największą liczbę ticketów;
- decyzje Foundera możliwe do zastąpienia zatwierdzoną polityką;
- lejek wiadomość -> formularz -> odpowiedź -> zastosowany efekt;
- niezależność i siła EQL;
- kompletność i świeżość Digital Twin;
- powtarzalne problemy DNS, TLS, Plesk i publikacji;
- koszt tokenów, API, kliknięć, przypomnień i czasu człowieka;
- wspólna logika występująca w co najmniej trzech modułach.

WIP auto-improvement powinien być ograniczony, przykładowo do jednej zmiany
krytycznej i dwóch zwykłych. Jeden fingerprint może mieć najwyżej jeden
aktywny ProblemCase i jeden Improvement Candidate.
