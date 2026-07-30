---
{
  "schema": "subactor.doc/v1",
  "id": "docs.architecture.autonomy-loop-and-twin",
  "version": 1,
  "status": "current",
  "updated": "2026-07-30"
}
---

# Pętla autonomiczna i twin: jak to działa i gdzie się rwie

Dokument opisuje zmierzony stan na 2026-07-30, nie stan docelowy. Każda liczba
pochodzi z obserwacji produkcyjnej instancji, nie z deklaracji w kodzie.

## 1. Co pętla robi dzisiaj

Cykl kontrolera (`platform/scripts/run-controller-cycle.mjs` → `POST
/api/tickets/lifecycle/reconcile`) skanuje aktywne tickety i klasyfikuje je na
`execute` / `notify` / `blocked` / `ignore`. Rekoncyliacja projektów
(`runProjectReconciliation`, `server.mjs`) osobno przelicza 8 projektów
domenowych i utrzymuje dla każdego ticket stanu.

Zmierzona skuteczność:

```
tickety ogółem 52 : open 32 | done 14 | failed 6
tickety automatyczne 36 : done 14  (39%)
wykonania procesów : succeeded 16 | failed 5
```

Pętla **realnie wykonuje i domyka** tickety. Problem leży gdzie indziej.

## 2. Gdzie się rwie: eskalacja jest jedyną gałęzią po porażce

```
otwarte wg blokera                          otwarte wg kolejki
waiting_input                19             boty       23
mutation_gate_disabled        4             founder     8
exact_route_missing           3
application_route_not_ready   1
```

**21 z 28 ticketów w stanie `waiting_input` siedzi w kolejkach botów**, mediana
wieku 6,4 dnia, maksimum 9,6. To nie są tickety, które czekają na decyzję
człowieka z projektu — to boty, które napotkały nieznaną sytuację i zatrzymały
się, bo nie mają czego zrobić poza eskalacją.

Trzy klasy blokerów kończą się eskalacją, choć wszystkie trzy są odpowiadalne
z danych, które platforma już posiada:

| Bloker | Pytanie, na które nikt nie odpowiada | Źródło odpowiedzi |
|---|---|---|
| `exact_route_missing` | czy ta trasa istnieje? jak się naprawdę nazywa? | snapshot bindings |
| `bridge_422:*_invalid` | jakie parametry przyjmuje ta trasa? | `inputSchema` w snapshocie |
| `capability_missing` | czy connector serwuje trasę, której nikt nie zarejestrował? | detektor dryfu |

## 3. Dlaczego twin tego nie podpowiadał

Twin i warstwa strategii planują **wyłącznie po katalogu zdolności**
(`platform/config/connector-capabilities/catalog.v1.json`, ładowany w
`core/services/control/src/plesk-capability-inventory.mjs`). Nigdy nie sięgają
do kodu connectora.

Katalog był pisany ręcznie i rozjechał się: `urirun-connector-plesk` serwował
**39 tras, zarejestrowanych było 17**. Wśród 22 niewidocznych był
`auth/command/bootstrap-api-key`, który wymienia login admina z sejfu na klucz
REST API. Ponieważ twin go nie widział, rozwiązywalny błąd 401 eskalował przez
kilka dni jako „brak poświadczeń Plesk" — zamiast „uruchom bootstrap".

Asymetria była w kodzie dosłowna:

- **stan środowiska** jest twinem: `subscription/query/snapshot` zwraca
  `twin_fact` ze `schema`, `snapshot_hash`, `freshness_ms`, `fact_quality`;
- **powierzchnia działań** nie była twinem w ogóle — jedyną autorytatywną listą
  były dekoratory `@conn.handler` w Pythonie, czytelne tylko wewnątrz procesu.

Dowód, że to nie jest problem teoretyczny: strategia wysłała
`{"apply": false}` bez `hostname` i dostała 422. Człowiek badający ten sam
system wysłał `{"domain": …}` tam, gdzie trasy chcą `site_id` / `host` /
`email`, i dostał 422 siedem razy. **Agent i człowiek zawiedli identycznie, bo
kontrakt parametrów nie był nigdzie publikowany.**

## 4. Co zostało zbudowane

### Twin powierzchni działań

`urirun` zamienia sygnaturę każdego handlera w schemat wejścia
(`conn.bindings()` → `urirun.bindings.v2`, 47 KB). Dane istniały, brakowało
konsumenta — żaden endpoint ich nie publikuje (`/bindings` → 404).

`platform/config/connector-capabilities/plesk.bindings.snapshot.json` to zrzut
tej powierzchni w konwencji istniejącej `plesk.doctor.fixture.json`: 39 tras,
dla każdej `effect` (query/command), `allow_execute`, etykieta i pełny kontrakt
parametrów (nazwa, typ, wartość domyślna, wymagalność), plus `snapshot_hash`.

### Detektor dryfu

`platform/scripts/audit-connector-capability-drift.mjs` porównuje snapshot
z katalogiem w obie strony:

- `capability_unregistered` — connector serwuje, katalog nie zna → niewidoczne
  dla autonomii;
- `capability_phantom` — katalog zna, connector nie serwuje → dyspozycja padnie.

Stan po naprawie: **39 serwowanych, 39 zmapowanych, 0 rozjazdów.**

### Gałąź research

`core/services/control/src/capability-surface.mjs` odpowiada na trzy pytania
z tabeli w §2, wyłącznie przez obserwację — nigdy nie dyspozycjonuje, więc jest
bezpieczna do uruchomienia *przed* decyzją, czy człowiek jest w ogóle potrzebny.

`researchBlocker()` zwraca `escalate: false` tylko wtedy, gdy research znalazł
coś wykonalnego. **`escalate: true` po researchu jest pełnoprawną odpowiedzią** —
znaczy „sprawdziliśmy i nic nie ma", co jest czymś innym niż „nigdy nie
spojrzeliśmy". Bramka celowo zamknięta (`mutation_gate_disabled`) nie jest
problemem researchu i nie wchodzi w tę gałąź.

## 5. Niezmiennik, którego łatwo nie zauważyć

Próba zamodelowania `tls_san_check` i `dns_preflight` jako „flag gotowości bez
trasy" (puste `uri_processes`) została odrzucona przez bramkę zdolności:
`capability_not_in_aql`.

**Zdolność, której nic nie potrafi wywołać, to zdolność, której żaden kontrakt
nie może autoryzować.** Pusta lista tras nie jest sposobem na modelowanie flagi
— trzeba wskazać trasy, które tę zdolność realnie dostarczają. Niezmiennik jest
teraz egzekwowany testem obejmującym wszystkie zdolności katalogu.

## 6. Cykl życia zadania: model rozszerzony

Szkielet `intent → build twin → test twin → do job → rollback` jest poprawny,
ale w zmierzonym stanie brakuje mu pięciu krawędzi. Poniżej wersja uzupełniona
o to, co dzisiejsze obserwacje wykazały jako konieczne.

```
intent (NL)
  │
  ├─ 1. OBSERVE ─────────────────────────────────────────────────────┐
  │     wyprowadź twin z rzeczywistości: stan (twin_fact) ORAZ        │
  │     powierzchnię działań (bindings snapshot).                     │
  │     Twin pisany ręcznie rozjeżdża się — 17/39 to dowód.           │
  │                                                                    │
  ├─ 2. BUILD TWIN                                                     │
  │     każdy fakt niesie snapshot_hash + freshness.                   │
  │                                                                    │
  ├─ 3. VERIFY TWIN ◄──────────────────────────────────────────────────┘
  │     detektor dryfu w obie strony. Twin, który przechodzi testy,
  │     ale rozjechał się z rzeczywistością, jest gorszy niż jego brak.
  │
  ├─ 4. TEST ON TWIN
  │     walidacja payloadu wobec inputSchema PRZED dyspozycją.
  │     Tu ginie {apply:false} i {domain:…} — bez ruchu sieciowego.
  │     Rollback jest tu trywialny, bo krok jest bezskutkowy z konstrukcji.
  │
  ├─ 5. DRY-RUN → plan_hash
  │     rehearsal na rzeczywistości bez mutacji; receipt niesie
  │     plan_hash ORAZ input_hash (co dokładnie wysłano).
  │
  ├─ 6. APPROVE(plan_hash)
  │     zgoda wiąże się z dokładnym hashem i wygasa po użyciu;
  │     apply weryfikuje, że hash się nie ruszył.
  │
  ├─ 7. DO JOB (pod dzierżawą)
  │     session/command/mutate-lease bramkuje mutacje. W pętli async
  │     dzierżawa jest tym, co nie pozwala dwóm cyklom działać naraz.
  │
  ├─ 8. VERIFY FROM OUTSIDE
  │     niezależny odczyt z zewnątrz hosta, nie „connector zwrócił 200".
  │
  └─ 9. ON FAILURE — trzy wyjścia, nie jedno
        ├─ RETRY   gdy input_hash się zmienił (przyczyna usunięta)
        ├─ RESEARCH gdy bloker jest researchowalny (§2) → wróć do 4
        └─ ESCALATE gdy research nic nie znalazł — z dowodem, co sprawdzono

        ROLLBACK zawodzi → człowiek, bezwarunkowo. Ta jedna gałąź
        nie może być autonomiczna.
```

Pięć krawędzi, których brakowało:

1. **OBSERVE przed BUILD** — nie da się zbudować twina środowiska, którego się
   nie zaobserwowało. Twin ręczny to nie twin, tylko notatka.
2. **Twin działań, nie tylko stanu** — bez `inputSchema` krok „test twin" nie
   ma czego sprawdzić i przepuszcza payload, który padnie na 422.
3. **VERIFY TWIN (dryf)** — świeżość i hash muszą być porównywalne
   z rzeczywistością, inaczej rehearsal jest przeciw fikcji.
4. **`input_hash` obok `plan_hash`** — `plan_hash` mówi „co zrobimy",
   `input_hash` mówi „co dokładnie wysłaliśmy". Bez tego drugiego porażka
   zatrzaskuje się na stałe i nie odblokuje jej nawet naprawa przyczyny.
5. **RESEARCH jako trzecie wyjście** — dziś porażka ma tylko rollback
   i eskalację, stąd 21 ticketów botów stojących średnio 6,4 dnia.

## 7. Co nadal jest otwarte

- Gałąź research istnieje jako moduł z testami, ale **nie jest jeszcze wpięta**
  w `ensureProjectRemediationTickets` — tickety wciąż eskalują bez niej.
- Snapshot bindings powstaje ręcznie; nie ma procesu, który by go odświeżał
  z zainstalowanego connectora (`--print-refresh-command` podaje komendę).
- Trasy `repo://workspace/uri-blueprints/*/command/implement` są autoryzowane
  kontraktem `project-operator`, ale żaden connector ich nie serwuje — to
  lustrzane odbicie problemu duchów, po stronie schematu `repo://`.
- `check-uri-collisions` zgłasza `capability_route_missing` dla **każdej**
  zdolności (0 zarejestrowanych tras globalnie), więc to ostrzeżenie jest
  szumem, nie sygnałem.
