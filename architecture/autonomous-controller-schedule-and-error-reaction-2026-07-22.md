---
{
  "schema": "subactor.doc/v1",
  "id": "docs.architecture.autonomous-controller-schedule-and-error-reaction-2026-07-22",
  "version": 5,
  "status": "current",
  "updated": "2026-07-22"
}
---

# Harmonogram kontrolera i reakcja na błędy

Data stanu: 2026-07-22. Dokument opisuje, **kiedy** uruchamia się kontroler
autonomicznej kolejki, **gdzie** jest jego konfiguracja, oraz **co dokładnie**
dzieje się po wystąpieniu błędu. Zawiera wynik kontrolowanego testu
wykonanego na działającym stosie.

## Wynik w jednym zdaniu

Pętla `błąd → wykrycie → deduplikacja → klasyfikacja → eskalacja → ticket →
wykonanie → completion receipt z asercjami EQL` **domyka się i została
zweryfikowana end-to-end**. Domyka się jednak do **zweryfikowanej diagnozy**,
nie do naprawy — pack reakcji ma etykietę `diagnosis-only` i
`automatic_mutation_allowed: false`.

## Gdzie jest konfiguracja

Nie ma crona ani jednostki systemd. Kontroler to pętla wewnątrz procesu
`hr-control`, zadeklarowana w `core/services/control/src/server.mjs`:

```
:2092   setTimeout(autonomousQueueConsumerCycle, AUTOMATION_STARTUP_GRACE_MS)
:2093   setInterval(autonomousQueueConsumerCycle, AUTONOMOUS_QUEUE_CONSUMER_INTERVAL_MS)
```

Pokrętła czytane ze środowiska (`server.mjs:154-162`):

| zmienna | domyślnie | walidacja przy starcie |
|---|---|---|
| `AUTONOMOUS_QUEUE_CONSUMERS_ENABLED` | `1` | — |
| `AUTONOMOUS_QUEUE_CONSUMER_INTERVAL_MS` | `30000` | min 10 000; poniżej start pada `autonomous_queue_consumer_interval_invalid` |
| `AUTONOMOUS_QUEUE_CONSUMER_LIMIT` | `5` | zakres 1–20; poza nim `autonomous_queue_consumer_limit_invalid` |
| `AUTOMATION_STARTUP_GRACE_MS` | `30000` | zakres 0–300 000; poza nim `automation_startup_grace_invalid` |
| `AUTONOMOUS_QUEUE_ERROR_TRIGGER_ENABLED` | `1` | — |
| `AUTONOMOUS_QUEUE_ERROR_TRIGGER_DELAY_MS` | `1000` | zakres 100–60 000; poza nim `autonomous_queue_error_trigger_delay_invalid` |

> **Luka konfiguracyjna — ZAMKNIĘTA 2026-07-22 20:17.** Wszystkie sześć kluczy
> istniało w `platform/.env.example`, ale **żadnego nie było w
> `platform/.env`**: stos chodził na domyślnych z kodu, a operator szukający
> harmonogramu w `.env` niczego tam nie znajdował.
>
> **Nie edytuje się `.env` ręcznie.** `scripts/validate-env.mjs` woła
> `initializeEnv({templateFile, contractFile, targetFile})`, które
> automigruje brakujące klucze z kanonicznego kontraktu
> (`config/env-contract.json`) i raportuje `Environment auto-migrated: added N
> variables.` Wystarczy:
>
> ```bash
> cd platform && node scripts/validate-env.mjs   # albo: make env-check
> ```
>
> Dopisało 20 zmiennych, w tym wszystkie sześć powyższych, z wartościami
> identycznymi jak domyślne w kodzie — czyli **bez zmiany zachowania**.
> Zweryfikowane: bramka CI zielona, kontener `hr-control` zdrowy.
>
> Jeden skutek uboczny wart odnotowania: migracja wnosi też
> `ANALYTICS_SERVICE_TOKEN=__GENERATE_ANALYTICS_TOKEN__`. Przed migracją `.env`
> nie miał ani jednego placeholdera, po niej ma jeden. Jest bezpieczny —
> w `docker-compose.yml` nie ma usługi `analytics`, a żaden kod nie czyta tej
> zmiennej — ale gdy taka usługa powstanie, token trzeba wygenerować, zanim
> ruszy.

### Wszystkie pętle czasowe w control

| linia | pętla | odstęp |
|---|---|---|
| `:2057` | cykl delegacji (anonimowa, własna bramka czasowa) | stała w kodzie |
| `:2082` | konsument autonomicznej kolejki | `AUTONOMOUS_QUEUE_CONSUMER_INTERVAL_MS` |
| `:2089` | dzienny digest foundera | `FOUNDER_DAILY_DIGEST_INTERVAL_MS` |
| `:2162` | pilne sprawy foundera | `FOUNDER_URGENT_INTERVAL_MS` |
| `:2182` | watchdog zawieszonych wykonań | `STALE_EXECUTION_WATCHDOG_INTERVAL_MS` |
| `:2190` | rekoncyliacja projektów | `PROJECT_RECONCILIATION_INTERVAL_MS` |

Numery linii w `server.mjs` przesuwają się szybko — plik urósł o ~50 linii w
ciągu jednej sesji. Szukaj po nazwie funkcji (`autonomousQueueConsumerCycle`,
`founderDailyDigestCycle`, …), a nie po numerze.

### „Raz na dzień" to nie odstęp, tylko bramka zegarowa

Kontroler kolejki **nie chodzi raz dziennie — chodzi co 30 sekund**, więc
wymóg dzienny jest spełniony z dużym zapasem.

Raz dziennie chodzi **digest foundera**, i jest zbudowany inaczej, niż
sugeruje nazwa zmiennej: `FOUNDER_DAILY_DIGEST_INTERVAL_MS=60000` to jedynie
**tyknięcie co minutę**, a o faktycznym wystrzale decyduje bramka zegarowa.
Te klucze **są** obecne w `platform/.env` (175–179):

```
FOUNDER_DAILY_DIGEST_ENABLED=true
FOUNDER_DAILY_DIGEST_HOUR=8
FOUNDER_DAILY_DIGEST_MINUTE=0
FOUNDER_DAILY_DIGEST_TIMEZONE=Europe/Warsaw
```

Mylenie `INTERVAL_MS` z częstotliwością zdarzenia jest tu najłatwiejszym
błędem — zmiana `INTERVAL_MS` na 86 400 000 nie zrobi „raz dziennie", tylko
sprawi, że bramka 08:00 będzie sprawdzana raz na dobę i najpewniej ją
przegapi.

## Ścieżka reakcji na błąd

Wyzwolenie jest **zdarzeniowe na całej długości**. Do 2026-07-22 18:15 druga
połowa była odpytywana (ticket czekał na najbliższy cykl 30-sekundowy);
`core/services/control/src/autonomous-controller-trigger.mjs` zamienił to na
wyzwalanie zdarzeniem. Obie wersje zmierzono — patrz „Test kontrolowany".

```text
błędna odpowiedź HTTP z control
  └─ observeProblemResponses (server.mjs:1993)
       └─ journalProblem (server.mjs:1984)
            ├─ audit "problem.detected"
            └─ recordProblemReaction  → grupa wg fingerprintu, licznik w oknie
                 └─ audit "problem.classified"  (strategia + authority)

            └─ shouldTriggerAutonomousController (autonomous-controller-trigger.mjs)
                 │   true gdy reaction.ticket_required === true
                 │   albo severity ∈ {error, fatal}
                 └─ cykl kontrolera po debounce AUTONOMOUS_QUEUE_ERROR_TRIGGER_DELAY_MS

cykl kontrolera (wyzwolony zdarzeniem albo planowy co 30 s)
  └─ ensureProblemReactionTickets (server.mjs:1417)
       └─ problemReactionTicketCandidates  → wymaga reaction.ticket_required === true
            └─ POST /tickets  (pack problem.reaction.observer, limit 1 na cykl)

NASTĘPNY cykl (celowo nie ten sam)
  └─ runAutonomousQueueCycle → claim → execute → completion receipt + EQL
```

Wyzwalacz nie zastępuje harmonogramu, tylko go wyprzedza: `setInterval` nadal
tyka co 30 s jako siatka bezpieczeństwa, a zdarzenie skraca oczekiwanie do
`delayMs`. Debounce jest konieczny, bo pojedynczy incydent potrafi wygenerować
serię błędów — bez niego każdy z nich budziłby osobny cykl.

Zwrotka: **każda** błędna odpowiedź HTTP z control staje się `problem.detected`.
Stąd wolumen — 1595 wykryć w dzienniku audytu z 26 godzin. Skąd naprawdę
pochodziły, opisuje osobna sekcja niżej.

Odcisk palca to `sha256(code, component, environment, resource)`
(`runtime/src/problem-details.mjs:193`), gdzie `resource` to ścieżka URL bez
query. Dwa błędy tego samego rodzaju na tej samej ścieżce trafiają do jednej
grupy.

Świeżo utworzony ticket reakcji **celowo nie jest wykonywany w tym samym
cyklu** (`server.mjs:1418-1420`) — Planfile ma czas utrwalić envelope, a to
daje naturalny backpressure. Zmierzone opóźnienie od eskalacji do utworzenia
ticketu wynosi ~2 s (wyzwalacz), a wykonanie następuje w kolejnym cyklu.

## Polityka klasyfikacji

Progi z `platform/config/problem-strategies/catalog.v1.json`:

```
window_ms                300000   (5 minut)
retry_base_ms             30000
retry_max_ms             900000
retry_escalation_count         3
access_escalation_count        5
max_groups                  1000
```

Strategie w kolejności priorytetu (wygrywa najwyższy):

| prio | strategia | warunek | `ticket_required` |
|---|---|---|---|
| 1000 | `problem.security.contain` | `problem.security == true` | — |
| 900 | `problem.dependency.escalate` | kategoria `dependency`/`rate_limit` + próg | — |
| 800 | `problem.dependency.retry` | kategoria `dependency`/`rate_limit`, retryable | `false` |
| 700 | `problem.internal.diagnose` | kategoria `internal` **lub** severity `error`/`fatal` | `true` |
| 600 | `problem.access.review` | kategoria `authentication`/`authorization` **oraz** `occurrences >= 5` | `true` |
| 0 | `problem.observe` | fallback: istnieje fingerprint | `false` |

Kluczowa konsekwencja: **HTTP 404 nigdy nie eskaluje sam z siebie.** Ma
kategorię `not_found` i severity `warn` (`runtime/src/problem-details.mjs:85-87`),
więc nie łapie się na żaden warunek poza fallbackiem. Dlatego dominującą
klasyfikacją jest `problem.observe` — to zachowanie zgodne z polityką, nie
zator.

## Test kontrolowany — przebieg i wynik

Test wykonano na działającym stosie 2026-07-22. Metoda: pięć
nieuwierzytelnionych żądań `GET http://127.0.0.1:8091/api/delegation/manager`
(HTTP 401 → kategoria `authentication`). Żadnej mutacji, żadnej zmiany
konfiguracji.

**Przewidywanie sformułowane przed uruchomieniem**, wyłącznie na podstawie
katalogu strategii: wystąpienia 1–4 dadzą `problem.observe`
(authority `component-owner`), wystąpienie 5 przekroczy
`access_escalation_count` i da `problem.access.review`
(authority `security`).

### Obserwacja 1 — klasyfikacja i próg

```
18:04:57 | occ_okno: 1 | occ_total: 3 | problem.observe        | authority:component-owner
18:05:25 | occ_okno: 2 | occ_total: 4 | problem.observe        | authority:component-owner
18:05:26 | occ_okno: 3 | occ_total: 5 | problem.observe        | authority:component-owner
18:05:27 | occ_okno: 4 | occ_total: 6 | problem.observe        | authority:component-owner
18:05:28 | occ_okno: 5 | occ_total: 7 | problem.access.review  | authority:security
```

Przewidywanie trafione co do jednego żądania. Potwierdzone jednocześnie:
deduplikacja do jednej grupy, licznik w oknie rosnący 1→5 niezależnie od
licznika całkowitego (3→7), próg wyzwalający się dokładnie na piątym
wystąpieniu, oraz eskalacja authority.

Reakcja po eskalacji:

```
ticket_required            : true
notification_due           : true
automatic_mutation_allowed : false
fingerprint                : 48537a5a89fb72ce…
```

### Obserwacja 2 — trzy kolejne cykle kontrolera

```
18:05:30 | scanned 992 | considered 46 | executable 0 | executed 0 | reaction_tickets_created: 1
18:06:16 | scanned 993 | considered 47 | executable 1 | executed 1 | reaction_tickets_created: 0
18:06:29 | scanned 993 | considered 46 | executable 0 | executed 0 | reaction_tickets_created: 0
```

Cykl bezpośrednio po eskalacji utworzył ticket i **go nie wykonał** — zgodnie
z zaprojektowanym backpressure. Wykonanie nastąpiło w cyklu następnym, po
czym ticket zniknął z puli rozważanych.

### Obserwacja 3 — co ticket faktycznie zrobił

Ticket `PLF-1046`, „Diagnose problem 48537a5a89fb — subactor.auth.authentication…",
etykiety `problem-reaction`, `diagnosis-only`. Start 18:05:59, koniec 18:06:11
(12 sekund), status `done`, aktor `safety-operator-bot`.

Cztery procesy URI, wszystkie OK:

```
read      problem://events/query/by-fingerprint
record    problem://reaction/command/record-occurrence
classify  problem://reaction/query/classification
audit     audit://problem/command/append-classification
```

Completion receipt, **6/6 asercji EQL zdanych**, każda z cytowanym dowodem:

```
PASS canonical-problem   verifier: process-step-receipt
PASS deduplicated        verifier: process-step-receipt
PASS deterministic       verifier: process-step-receipt
PASS observation-only    verifier: process-step-receipt
PASS governed-followup   verifier: process-step-receipt
PASS safe-evidence       verifier: process-step-receipt
```

> **Pułapka przy weryfikacji.** Asercje leżą w
> `outputs.completion_receipt.eql`, a nie w `outputs.assertions` ani
> `outputs.result.assertions`. Odczyt z niewłaściwego pola zwraca pustą listę
> i wygląda jak brak weryfikacji Definition of Done, choć weryfikacja
> przebiegła poprawnie.

### Runda druga — pomiar wyzwalacza zdarzeniowego

Runda pierwsza (18:04–18:06) wypadła na buildzie **sprzed** wyzwalacza;
kontener został zrestartowany o 18:15:31 z `autonomous-controller-trigger.mjs`.
Test powtórzono na nowym buildzie, tą samą metodą, z pomiarem czasu:

```
18:17:38.4   żądanie 1                        → problem.observe   (okno 1)
18:17:39.5   żądanie 2                        → problem.observe   (okno 2)
18:17:40.6   żądanie 3                        → problem.observe   (okno 3)
18:17:41.8   żądanie 4                        → problem.observe   (okno 4)
18:17:42.9   żądanie 5                        → problem.access.review (okno 5)
18:17:45.0   cykl kontrolera → reaction_tickets_created: 1
18:18:01     PLF-1048 start wykonania
18:18:14     PLF-1048 done, 6/6 EQL, 4 procesy URI
18:18:19.1   cykl odnotowuje executed: 1
```

**Opóźnienie od eskalacji do utworzenia ticketu: 2,14 s.** Poprzedni cykl
zakończył się o 18:17:31.8, więc planowy następny wypadałby ok. 18:18:01 —
cykl ruszył **16 sekund przed terminem**. To jest dowód działania wyzwalacza,
a nie trafu: przy samym `setInterval` ticket czekałby do 18:18:01.

Porównanie obu rund (ten sam odcisk palca `48537a5a89fb`, ta sama ścieżka):

| | runda 1 (bez wyzwalacza) | runda 2 (z wyzwalaczem) |
|---|---|---|
| próg eskalacji | 5. wystąpienie | 5. wystąpienie |
| eskalacja → ticket | do 30 s (traf: 2 s) | **2,14 s, deterministycznie** |
| wykonanie ticketu | cykl następny | cykl następny |
| asercje EQL | 6/6 | 6/6 |
| ticket | PLF-1046 | PLF-1048 |

Próg, deduplikacja i weryfikacja zachowały się identycznie — zmieniło się
wyłącznie opóźnienie reakcji, i zmieniło się z „do 30 sekund" na
przewidywalne 2 sekundy.

## Co z tego wynika

Działa i jest udowodnione:

- kontroler żyje i tyka co 30 s (1249 zapisów `autonomous_queue.cycle.completed`),
  a wyzwalacz zdarzeniowy wyprzedza harmonogram w ~2 s po eskalacji;
- deduplikacja po odcisku palca z oknem 5 minut;
- progi eskalacji egzekwowane co do jednego wystąpienia;
- deterministyczny wybór strategii wg priorytetu;
- eskalacja tworzy ticket, kolejny cykl go wykonuje, wynik ma receipt z
  asercjami EQL i dowodami;
- zakaz automatycznej mutacji utrzymany na całej ścieżce.

Nie działa, albo działa inaczej, niż można oczekiwać:

1. **Pętla domyka się do diagnozy, nie do naprawy.** Pack ma etykietę
   `diagnosis-only`. Nie istnieje krok „diagnoza → naprawa" i to jest
   świadoma granica, a nie usterka — ale trzeba ją znać, zanim ktoś uzna, że
   system „sam się naprawia".
2. **~~1577 wykryć zdominowanych przez nierozwiązany upstream.~~** Pierwsza
   wersja tego dokumentu przypisywała wolumen blockerowi PLF-592. **To był
   domysł i był błędny** — sprawdzone i sprostowane, patrz „Skąd wziął się
   wolumen wykryć" niżej. Krótko: 91,8 % pochodziło z jednego wewnętrznego
   endpointu, a przyczyna została naprawiona commitem `3a7dfa7` o 14:47.
3. **Harmonogram jest niewidoczny w `.env`** — patrz luka konfiguracyjna
   wyżej.
4. **46 ticketów nieterminalnych, 0 wykonywalnych.** To poprawne zachowanie
   fail-closed: 43 czekają na wejście człowieka, 3 nie mają pasującej reguły
   i spadają na foundera. Kontroler odmawia zgadywania właściciela.

## Skąd wziął się wolumen wykryć — dochodzenie i sprostowanie

Pierwsza wersja tego dokumentu stwierdzała, że 1577 wykryć „wygląda na
nierozwiązany blocker PLF-592 mielony w kółko". **To był domysł, nie pomiar, i
okazał się nieprawdziwy.** Poniżej faktyczny rozkład.

### Pomiar

Z 1595 wpisów `problem.detected` obejmujących 2026-07-21 18:18 → 2026-07-22 20:00:

| udział | zasób |
|---|---|
| **1465 (91,8 %)** | `/api/internal/integrations/resolve` |
| 13 (0,8 %) | `/api/founder/delegations/respond` |
| 12 (0,8 %) | `/api/projects/reconciliation/run` |
| 12 (0,8 %) | `/api/processes/run` |
| 12 (0,8 %) | `/api/delegation/manager` |

Unikalnych odcisków palca: 52. PLF-592 nie występuje w czołówce w ogóle.

### Mechanizm

Wykrycia przychodziły **seriami po pięć w ciągu milisekund**. Źródłem jest
`connectors/services/bridge/src/server.mjs:1387`:

```js
const [email, slack, teams, webhook] =
  await Promise.all(["email", "slack", "teams", "webhook"].map(dynamicProfile));
```

Cztery rozwiązania równolegle plus wcześniejsze `plesk` — dokładnie pięć.
Każde woła `/api/internal/integrations/resolve`. Gdy integracja nie jest
skonfigurowana, control odpowiadał 404 `integration_not_found`, a
`observeProblemResponses` zapisywało to jako problem. Brak skonfigurowanej
integracji opcjonalnej jest **normalnym stanem**, więc telemetria pokazywała
tysiąc zdarzeń o niczym.

### Naprawa i weryfikacja

Naprawione commitem `3a7dfa7` „fix(connectors): preserve evidence after
external effects" (2026-07-22 14:47):

```diff
-    resolveDynamicIntegration(null, {system_provider: provider})
+    resolveDynamicIntegration(null, {system_provider: provider, optional: true})
```

Control ma dla tego jawną gałąź (`routes/integrations.mjs:16`): gdy
`optional === true` i nie podano `integration_id`, zwraca **200** z
`integration: null` zamiast 404. Zweryfikowane bezpośrednio na wdrożonym
kodzie:

```
{system_provider:"plesk", optional:true}          -> 200  {"integration": null}
{system_provider:"plesk"}                         -> 404  integration_not_found
{integration_id:"nope", optional:true}            -> 404  integration_not_found
```

Rozkład godzinowy potwierdza skutek:

```
07-21 19:00   716   ← szczyt
07-21 21:00    40
07-22 06:00    72
07-22 07:00    35
07-22 08:00+    0   ← koniec
```

Po 07:25 nie ma ani jednego wpisu z tego zasobu. Dwa widoczne o 20:00:48 to
artefakty powyższej sondy weryfikacyjnej — celowe 404 wywołane, żeby
udowodnić, że gałąź `optional` działa.

### Co z tego zostaje

- **Bieżąca telemetria jest czysta.** Liczba 91,8 % jest prawdziwa
  historycznie i myląca jako opis stanu na dziś.
- Trzeci wiersz tabeli weryfikacyjnej pokazuje, że `optional: true` **nie**
  tłumi 404, gdy podano konkretne `integration_id`. To wygląda na decyzję
  celową — nazwanie konkretnej integracji, której nie ma, jest błędem, a nie
  brakiem opcjonalnej konfiguracji. Zostawione bez zmian.
- Wniosek ogólny: przy tej architekturze **każdy nowy wewnętrzny endpoint
  zwracający 4xx dla stanu normalnego zamienia się w telemetryczny szum**.
  Warto to sprawdzać przy dodawaniu tras, a nie po fakcie w dzienniku.

## Jak powtórzyć test

```bash
# 1. stan przed
docker exec subactor-platform-hr-control-1 sh -c \
  'grep -c "\"problem.classified\"" /data/audit.jsonl'

# 2. pięć nieuwierzytelnionych żądań (401 → kategoria authentication)
for i in 1 2 3 4 5; do
  curl -s -o /dev/null -w "%{http_code}\n" \
    http://127.0.0.1:8091/api/delegation/manager
  sleep 1
done

# 3. klasyfikacje — oczekiwane: 4× problem.observe, potem problem.access.review
docker exec subactor-platform-hr-control-1 sh -c \
  'grep "\"problem.classified\"" /data/audit.jsonl | tail -5'

# 4. cykle kontrolera — oczekiwane: reaction_tickets_created 1, potem executed 1
docker exec subactor-platform-hr-control-1 sh -c \
  'grep autonomous_queue.cycle.completed /data/audit.jsonl | tail -3'
```

Test jest bezpieczny: nie mutuje stanu, nie dotyka DNS, Pleska ani sekretów.
Tworzy jeden ticket diagnostyczny, który sam się zamyka.

## Kontrakty i źródła

- harmonogram: `core/services/control/src/server.mjs:152-157, 2039-2040`
- ścieżka reakcji: `core/services/control/src/server.mjs:1984-1993, 1413-1450`
- klasyfikacja: `core/services/control/src/problem-reaction.mjs`
- bramka ticketu: `core/services/control/src/problem-reaction-ticket.mjs:19-46`
- polityka i strategie: `platform/config/problem-strategies/catalog.v1.json`
- odcisk palca i katalog problemów: `runtime/src/problem-details.mjs:85-107, 193-200`
- pack: `platform/config/process-packs/problem-reaction-observer/`
- asercje: `core/services/control/src/autonomous-queue-controller.mjs`
  (`completionAssertionsForTicket`)
