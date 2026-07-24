---
{
  "schema": "subactor.doc/v1",
  "id": "docs.architecture.adr.010-organization-constitution-and-ticket-currency",
  "version": 12,
  "status": "current",
  "updated": "2026-07-23"
}
---

# ADR-010: konstytucja organizacji i aktualność ticketów

- **Status:** Accepted
- **Data:** 2026-07-23
- **Konstytucja:** `subactor.organization-constitution/v1`
- **Aktualność ticketu:** `subactor.ticket-currency/v1`

## Decyzja

Najwyższym normatywnym obiektem systemu jest wersjonowana **Organization
Constitution**. Wiąże organizację, root authority, immutable actor contract,
model akceptacji, rejestr lifecycle oraz niezmienne zasady bezpieczeństwa.
Intencja pozostaje najwyższym semantycznym „dlaczego”, ale nie może sama nadać
uprawnień.

Aktywna instancja znajduje się w
`platform/config/governance/organization-constitution.v1.json`. Control
waliduje ją przy starcie i nie uruchamia się z błędnym hashem, niezatwierdzoną
aktywną wersją albo nieprawidłowym root contract.

```text
 wypowiedź człowieka / zdarzenie
              |
              v
        Intent Contract
              |
              v
 Organization Constitution ------> root AQL authority
              |                          |
              v                          v
       lifecycle registry       delegation constraints
              |                          |
              +------------+-------------+
                           v
                ticket + Process Envelope
                           |
             +-------------+-------------+
             v                           v
    readiness / currency            AQL + OQL + URI
             |                           |
             +-------------+-------------+
                           v
                    EQL + receipts
```

## Dwa niezależne pytania o ticket

`readiness` i `currency` nie mogą być jednym polem:

- **readiness** odpowiada, czy ticket można teraz bezpiecznie wykonać: status,
  wersja kontraktu, Process Envelope v2, AQL/EQL/OQL, graf kroków, exact URI,
  aktor i budżet prób;
- **currency** odpowiada, czy ticket nadal reprezentuje aktualną pracę:
  jawny następca, jawne źródło, bieżący observed state i dowody;
- audyt LLM może pomóc ocenić znaczenie dowodów, ale nie może sam wykonać
  mutacji;
- wiek, podobna nazwa albo historyczna notatka nie wystarczają do anulowania.

Deterministyczny kontrakt `subactor.ticket-currency/v1` korzysta tylko z
formalnych relacji:

- `source:PLF-N` — praca pochodna jest aktualna tak długo, jak źródło pozostaje
  aktywne;
- `superseded-by:PLF-N` — jawny następca unieważnia poprzedni ticket;
- brak obserwowanego ticketu relacji daje `uncertain`, nigdy automatyczne
  anulowanie.

Ticket z terminalnym źródłem albo aktywnym jawnym następcą dostaje
`cancel_recommended=true` i nie może zostać wykonany przez konsumenta kolejki.
W bieżącej wersji kontroler raportuje rekomendację i zapisuje audit event;
faktyczne anulowanie pozostaje oddzielnym, jawnym apply audytora.

## Odpowiedzialność walidatorów

- **Planfile** jest źródłem stanu i historii. Model `TicketInputs` odrzuca
  nieznane pola, Process Envelope v2 wymaga kompletnych definicji
  AQL/EQL/OQL/URI, a endpoint `complete` wymaga zgodnego completion receiptu
  z pozytywnymi asercjami EQL.
- **Runtime** dostarcza czyste, deterministyczne decyzje:
  `ticketReadinessDecision`, `ticketCurrencyDecision` oraz walidację bindingu
  konstytucji. Nie wykonuje mutacji.
- **Control** składa te decyzje z observed state, AQL aktorów i aktywnymi
  trasami URI. Może claimować, wykonywać, przypominać lub uzgadniać projekcję,
  lecz każda zmiana przechodzi ponownie przez kontrakt Planfile.
- **Audyt LLM** ocenia znaczenie bieżących i historycznych dowodów w trybie
  dry-run. Nie jest autorytetem stanu; bez jawnego `--apply`, progu confidence
  i rationale nie może wykonać zamknięcia.

## Constitution, Assurance i Controller

Constitution i Controller odpowiadają na inne pytania:

- **Organization Constitution** określa, skąd pochodzi authority, jakie
  delegacje są legalne i jakie inwarianty muszą obowiązywać niezależnie od
  aktualnej implementacji;
- **Autonomy Assurance Supervisor** ocenia, czy Controller, jego kontrakty i
  efekty nadal zachowują inwarianty. Nie powinien wykonywać zwykłych ticketów
  ani zatwierdzać własnych napraw;
- **Controller** prowadzi normalną pracę operacyjną: pobiera snapshot,
  sprawdza readiness i currency, uruchamia exact URI, zapisuje receipts,
  przypomina odpowiedzialnym aktorom i uzgadnia projekcje lifecycle;
- **zewnętrzny Sentinel** obserwuje heartbeat Control z innego procesu.
  Wewnętrzny scheduler nie może wiarygodnie wykryć własnego zatrzymania.

```text
 human intent / event / observed state
                  |
                  v
       Organization Constitution
          root authority + invariants
                  |
        +---------+----------+
        |                    |
        v                    v
 Autonomy Assurance      Controller
 DOQL -> DQL -> Doctor    ticket readiness/currency
          |               AQL -> OQL -> URI Process
          v                         |
 RepairCandidate                    v
          |                    EQL + receipts
          v                         |
 isolated canary                    |
          |                         |
          v                         |
 independent Validator <------------+
          |
    +-----+------------------+
    | green                 | red
    v                       v
 promotion eligible     rollback / retry /
 (no authority)         governed escalation

 External Sentinel -------- heartbeat / lease --------> Control
```

Warstwa Assurance składa istniejące języki zamiast tworzyć drugi język
wykonawczy:

1. SODL utrwala zdarzenia i przebieg.
2. DOQL agreguje zwalidowane obiekty do sytuacji.
3. DQL porównuje sytuację z wersjonowanymi inwariantami.
4. Doctor tworzy diagnozę i wymagania dowodowe, bez mutacji.
5. Strategy DSL wybiera dozwoloną klasę reakcji.
6. AQL ogranicza aktora i capability.
7. OQL oraz URI Process opisują i wykonują przypiętą naprawę.
8. niezależny Validator wystawia EQL/read-back receipt.

Zielony raport canary może nadać wyłącznie
`promotion_eligible=true`. Nie jest grantem, AQL ani dowodem wykonania w
produkcji.

## Inwarianty działania kontrolera

Sama obecność procesu i regularny heartbeat nie dowodzą autonomii. Assurance
musi osobno mierzyć:

- **liveness** — cykl kończy się w czasie wynikającym z konfiguracji;
- **progress** — pomiędzy kolejnymi oknami powstają receipts, terminalne
  rezultaty albo formalne powody oczekiwania;
- **safety** — brak wykonania bez aktualnego kontraktu, exact URI, AQL i EQL;
- **currency** — ticket nie ma terminalnego źródła ani aktywnego następcy;
- **uniqueness** — jeden fingerprint problemu i jeden aktywny RepairCandidate;
- **independence** — Repair nie może wystawić własnego pozytywnego dowodu;
- **recoverability** — czerwony Validator prowadzi do rollbacku albo jawnego
  stanu wymagającego interwencji;
- **coverage** — zielony DOQL/DQL ma receipt kompletności i świeżości każdego
  wymaganego źródła.

Brak postępu nie jest tym samym co awaria. Ticket czekający na człowieka,
zewnętrzny grant albo jawny ticket zależny może poprawnie nie zmieniać stanu.
Supervisor powinien uznać zastój dopiero wtedy, gdy minął SLA odpowiedzialności
i nie powstał receipt dostawy, odpowiedzi, retry lub eskalacji.

## Stwierdzone luki

Odczyt live z 2026-07-23 wykazał:

- cykl działa co 300000 ms, trigger błędów ma opóźnienie 1000 ms i cooldown
  300000 ms;
- 32 aktywne tickety były `currency=current`;
- 31 ticketów wymagało powiadomienia, jeden został zignorowany zgodnie z
  lifecycle, a żaden nie był wykonywalny;
- wszystkie 32 aktywne tickety miały `constitution_legacy_unbound`;
- aktywna konstytucja pozostaje w trybie `observe`.

Wykryto również dwie luki diagnostyczne:

1. live observer uznawał cykl za stary po stałych 120000 ms, mimo że prawidłowy
   okres kontrolera wynosi 300000 ms. Luka została usunięta: próg wynosi teraz
   `max(120000, 2 * periodic_interval_ms + jitter_budget_ms)`, a brak
   harmonogramu i clock skew są osobnymi naruszeniami;
2. profil DOQL portfela ocenia osiem lokalnych `project.manifest.json`.
   Zielone `8/8` dowodzi spójności tego zbioru, ale bez receiptu kompletności
   nie dowodzi spójności całego rejestru projektów Organization Core.

Kolejna walidacja restartu Control wykazała dwa przypadki, które wymagają
osobnych stanów zamiast wspólnego `stale`:

- przed `automation_ready_at` kontroler jest `startup_pending`;
- po rozpoczęciu pracy, lecz przed pierwszym receiptem, jest `cycle_running`;
- dopiero brak cyklu albo przekroczenie budżetu daje odpowiednio
  `controller_cycle_stale` lub `controller_cycle_timeout`.

Każdy proces Control otrzymuje niepowtarzalny `controller_instance_id`, a
każdy cykl `cycle_id`, `cycle_started_at`, `cycle_completed_at` i
`duration_ms`. Historia progress jest filtrowana do bieżącej instancji, dzięki
czemu restart nie miesza starych cykli z nowym procesem.

Cykl publikuje również
`subactor.controller-cycle-telemetry/v1`. Kontrakt ma zamknięty, uporządkowany
katalog etapów:

1. `snapshot_load`;
2. `problem_reactions`;
3. `readiness_recheck`;
4. `ticket_currency`;
5. `lifecycle_reconciliation`;
6. `reminder_preparation`;
7. `queue_execution`.

Nieznany, zduplikowany lub wykonany poza kolejnością etap jest odrzucany.
Każdy rekord zawiera wyłącznie status, czas i bezpieczną klasę błędu — nigdy
treść wyjątku, token ani payload. Control zapisuje timings w bieżącym
snapshotcie i completion/failure audit event, a Autonomy Lab weryfikuje
zgodność schematu, `cycle_id`, kompletność i sumę czasów.

Etap `problem_reactions` ma odrębny kontrakt
`subactor.problem-reaction-telemetry/v1`:

```text
candidate_selection
        |
        v
manifest_compilation
        |
        v
ticket_creation
        |
        v
audit_receipt
```

Gdy nie ma kandydata, pierwszy etap kończy się poprawnie, a trzy pozostałe
otrzymują jawny status `skipped` z powodem `no_candidate`. Nie są raportowane
jako wykonane i nie zwiększają progress. Kontrakt dopuszcza najwyżej jeden
utworzony ticket, zgodnie z istniejącym backpressure kontrolera.

DQL architektury sprawdza obecnie trzy inwarianty ze statycznego snapshotu.
Nie ma jeszcze live route registry ani automatycznego Finding Outbox
prowadzącego do `ProblemCase`. Doctor, Repair i Validator mają testowalne
implementacje, ale nie działają jeszcze jako niezależny recovery plane
odseparowany procesowo od Control.

## Plan domknięcia luk

### Etap 1 — poprawność obserwacji

- próg stale jest już wyliczany z konfiguracji:
  `max(120000, 2 * periodic_interval_ms + jitter_budget_ms)`;
- `controller_instance_id`, `cycle_id`, `cycle_started_at`,
  `cycle_completed_at`, `duration_ms` oraz siedem stage timings są już
  emitowane; nadal należy dodać lease ownera i histogram trendu wielu cykli;
- wymagać dla każdego źródła DOQL `freshness`, `expected_scope`,
  `observed_count`, hash snapshotu i receipt kompletności;
- bazowy profil `controller.progress` rozróżnia `productive`,
  `productive_with_waiting`, `waiting_external`, `idle` i
  `stalled_executable`; następna wersja ma dodać wiek najstarszego obowiązku,
  SLA odpowiedzialności oraz budżet retry/escalation.

### Etap 2 — skuteczna konstytucja

- wiązać wszystkie nowe tickety z `constitution_ref` i
  `constitution_hash`;
- zmigrować aktywny backlog i osobno oznaczyć historyczne legacy;
- uruchomić shadow comparison `observe` kontra `enforce`;
- przełączyć na `enforce` dopiero przy zerowej liczbie aktywnych
  `legacy_unbound` i zielonym canary;
- nie przepisywać historycznych ticketów bez receiptu migracji.

### Etap 3 — niezależny Assurance Plane

- uruchomić Supervisor, Doctor, Repair i Validator jako odrębne principals z
  rozłącznymi AQL;
- dodać idempotentny Finding Outbox:
  `DQL finding -> ProblemCase -> jeden ticket -> RepairCandidate`;
- przypinać snapshot, strategię i kandydata hashami;
- wykonywać najpierw symulację i izolowany canary;
- dopuścić produkcyjny URI Process dopiero po polityce promocji;
- wymagać niezależnego EQL/read-back i automatycznego rollbacku;
- uruchomić zewnętrzny Sentinel poza procesem i credentialami Control.

### Etap 4 — progresywne wdrożenie

Kolejność środowisk to: fixture, test jednostkowy, stress, izolowany canary,
shadow live, canary produkcyjny o odwracalnym skutku, a dopiero potem
ograniczona autonomia. Każdy etap ma osobny receipt i nie dziedziczy authority
z poprzedniego.

Szczegółowe komendy, oczekiwane wyniki i kryteria przerwania opisuje
`docs/operations/autonomy-assurance-validation-2026-07-23.md`.

## Rollout konstytucji

Tryb `observe` zachowuje kompatybilność istniejących ticketów:

- brak bindingu daje `legacy_unbound` i nie blokuje;
- jawny binding ze starym ref/hash daje `stale` i blokuje wykonanie;
- nowe ticket factories mają docelowo wiązać bieżący ref/hash;
- tryb `enforce` można włączyć dopiero po migracji aktywnego backlogu.

Formularze pochodne zapisują relację jako label `source:PLF-N`. Jest to pole
obsługiwane przez kontrakt Planfile i odczytywane przez walidator aktualności.
Kanoniczny magazyn formularzy zachowuje pochodzenie `ticket-derived:PLF-N`,
więc kontroler może idempotentnie uzupełnić label w istniejących rekordach.
Relacja nie jest duplikowana w `TicketInputs`, ponieważ ten obiekt celowo
odrzuca pola spoza wersjonowanego modelu.

## Uruchamianie i audyt

Control udostępnia uwierzytelniony
`POST /api/tickets/lifecycle/run` ze scope `routing:manage`. Endpoint uruchamia
ten sam współbieżnościowo chroniony cykl co scheduler i zwraca ograniczony
snapshot, bez omijania AQL, EQL, grantów ani master gate.

Aktualność biznesową sprawdza `platform/scripts/ticket-auditor.mjs`. Domyślnie
jest to dry-run. `duplicate`, `obsolete` i `done` mogą zmienić ticket dopiero z
`--apply`, po przekroczeniu progu confidence i z zapisem rationale oraz
receiptu.

Cykl musi używać jednego snapshotu operacyjnego. Powtarzanie pełnych odczytów
Planfile przez niezależne pętle lub nieidempotentna migracja relacji może
przeciążyć źródło prawdy. Migracja uznaje istniejący label za zakończony stan,
nie zapisuje ponownie dużego `inputs`, a równoległy cykl jest odrzucany przez
guard kontrolera.

Control współdzieli 30-sekundowy snapshot aktywnych ticketów pomiędzy
kontrolerem, przypomnieniami i projekcjami. Równoległe odczyty są koalescowane,
a każda mutacja unieważnia cache. Rekoncyliacja projektów jest procesem
dobowym, z ręcznym endpointem do kontrolowanego uruchomienia; nie konkuruje co
kilka minut z głównym cyklem autonomii.

Okresowy cykl autonomii działa co 5 minut, a niezależny trigger błędów może
uruchomić go wcześniej po kontrolowanym debounce. Startup digestu, pilnych
akcji Foundera i rekoncyliacji projektów ma osobne przesunięcia, aby nie
wykonywać kilku kosztownych snapshotów w tej samej sekundzie. Wszystkie
wartości są jawne w konfiguracji środowiska.

Rekoncyliacja projektów rozdziela obserwację od mutacji projekcji. Każdy wynik
trafia do audytu i snapshotu kontrolera, ale ticket projektu, ticket naprawczy
i ticket kontrolera są PATCH-owane tylko przy zmianie materialnej. Pola
czasowe `observed_at` i generowany `created_at` koperty nie powodują odnowienia
niezmienionego kontraktu; istniejąca aktualna koperta pozostaje niezmienna.

Trigger błędów koalescuje zdarzenia także w trakcie aktywnego cyklu. Po
zakończeniu obowiązuje 5-minutowy cooldown; symptomy z tego samego incydentu
uruchamiają najwyżej jeden kolejny cykl po cooldownie. Zapobiega to pętli
`timeout Planfile → problem.detected → pełny skan → kolejny timeout`, nie
usuwając reakcji zdarzeniowej.

Jednorazowy formularz ma odrębny lifecycle od ticketu źródłowego. Po
zwalidowaniu i zapisaniu odpowiedzi jego ticket dowodowy kończy się jako
`done`; dopiero potem jawna relacja `source:PLF-N` wskazuje, gdzie zastosować
decyzję. Błąd readiness ticketu źródłowego nie może ponownie otworzyć
formularza ani wygenerować przypomnienia o odpowiedzi, która już istnieje.
Kontroler uzgadnia historyczne rozbieżności tylko wtedy, gdy kanoniczny store
ma stan `completed` i udany submission.

## Dowody wdrożenia

- Runtime: testy hasha konstytucji, zakazu sekretów, observe/enforce oraz
  jawnego supersession.
- Core: test zatrzymania stale-bound i obsolete-derived ticketu, ręcznego
  cyklu oraz powiązania formularza ze źródłem.
- Gateway: zgodność kontekstu ticketu v2 z przejściową obsługą v1.
- Live dry-run 2026-07-23: 34 aktywne tickety, 34 `still_needed`, 0 błędów,
  0 kandydatów do zamknięcia.
- Pełny live dry-run 2026-07-23: 36 aktywnych ticketów, 36 `still_needed`,
  0 błędów i 0 kandydatów do anulowania.
- Cykl ustalony po wdrożeniu współdzielonego snapshotu: 52 tickety, 32
  `currency_current`, 0 `uncertain`, 0 `obsolete`, 0 zapisów naprawczych;
  odpowiedź API 63,7 ms, Planfile health 1,3 ms.
- Jednorazowa rekoncyliacja domknęła jako `done` cztery zwalidowane formularze:
  PLF-1105, PLF-1120, PLF-1127 i PLF-1167. Żaden ticket nie został anulowany.
- Live diagnostyka 2026-07-23: wykryto i usunięto nieidempotentny zapis relacji,
  który przez pole nieobsługiwane przez `TicketInputs` ponawiał PATCH i
  przeciążał Planfile.
- DQL architektury 2026-07-23: 3/3 inwarianty zielone.
- DOQL portfela manifestów: 8 obiektów, 8 domen, 8 unikalnych domen i brak
  kandydata decyzji; wynik dotyczy wyłącznie jawnie odczytanego zbioru.
- Autonomy Lab: 17/17 scenariuszy funkcjonalnych oraz 1700/1700 wykonań stress,
  bez mutacji zewnętrznych.
- Repair canary stress: 600/600 wykonań, w tym write failure, validator
  failure, candidate hash drift, LLM URI injection i duplicate execution;
  zero mutacji zewnętrznych.
- Testy Constitution, lifecycle registry i zakazu rekurencyjnej naprawy:
  7/7.
- Read-only lifecycle reconcile: 32 aktywne, 32 current, 32
  `legacy_unbound`, 31 notify, 1 ignore i 0 execute.
- Po poprawce live observer obliczył dla okresu 300 s i jitteru 30 s próg
  630 s. Cykl mający 218164 ms został poprawnie uznany za świeży: `ok=true`,
  zero violations i zero mutacji zewnętrznych.
- Autonomy Lab po zmianie: 9/9 testów, w tym granice fake-clock, brak
  harmonogramu, clock skew i okres przekraczający dobowe wymaganie.
- Zweryfikowane instancje Control po restarcie mają własny
  `controller_instance_id`; pierwszy zakończony cykl został zapisany jako
  `startup`, utworzył jeden deduplikowany ticket reakcji i nie wykonał efektu
  zewnętrznego. Najnowszy odczyt: 54 scanned, 33 considered, 0 executable,
  33 notifications i czas 2302 ms. Profil oznaczył go jako `productive`, a
  nie jako postęp wynikający z samych powiadomień.
- Live shadow podczas następnego pierwszego cyklu zwrócił
  `state=cycle_running`, poprawny `cycle_id`, brak violations oraz
  `external_mutations=0`.
- Autonomy Lab po dodaniu izolacji instancji, progress i stanu aktywnego cyklu:
  14/14. Celowane testy Core kontrolera: 23/23.
- Stage telemetry: Core 27/27, Autonomy Lab 16/16 i stress 1700/1700.
  Pierwszy live cycle miał 2627 ms czasu całkowitego i 2623 ms w
  instrumentowanych etapach. `problem_reactions` trwał 2523 ms, czyli około
  96% czasu etapów; sześć pozostałych etapów zajęło łącznie 100 ms.
- Pełny Core po integracji: 642 zaliczone, 7 pominiętych, 0 błędów.
- Etap utworzył dokładnie jeden ticket PLF-1189 dla nowego fingerprintu.
  Ostatnie tickety reakcji mają różne fingerprinty, więc obserwacja nie
  wskazuje na duplikację tego samego ProblemCase.
- Podtelemetria reakcji: Core celowane 29/29, Autonomy Lab 18/18, stress
  1700/1700 oraz pełny Core 644 zaliczone, 7 pominiętych, 0 błędów.
- Pierwszy live read-back po wdrożeniu miał `candidate_count=0`,
  `tickets_created=0`, `candidate_selection=completed` oraz trzy jawne
  `skipped:no_candidate`. Cały cykl miał 88 ms etapów, zero violations i zero
  mutacji zewnętrznych. Progress był poprawnie `waiting_external`.

## Konsekwencje

- Konstytucja jest nadrzędnym źródłem norm, ale nie zastępuje intencji,
  autorytetów domenowych ani observed state.
- Kontroler nie „zgaduje”, że ticket jest stary. Zatrzymuje wykonanie tylko na
  formalnym dowodzie nieaktualności.
- Formularz, wiadomość i ticket naprawczy muszą deklarować źródło; inaczej
  ogólny reconciler nie może bezpiecznie ustalić supersession.
- Automatyczne anulowanie wszystkich rekomendacji wymaga osobnego policy
  profilu, canary i rollbacku. Nie jest domyślnie włączone.
