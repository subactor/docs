---
{
  "schema": "subactor.doc/v1",
  "id": "docs.architecture.project-change-analysis-2026-08-04",
  "version": 11,
  "status": "current",
  "updated": "2026-08-04"
}
---

# Analiza zmian projektu — od Konstytucji do ticketu

## Cel

Project Composer przed materializacją zmian porównuje źródła wykonawcze z
hierarchią kierunku organizacji:

1. aktualne wskazanie użytkownika i wynikający z niego deterministyczny szkic;
2. aktywne oraz planowane tickety Planfile przypisane do `project_id`;
3. przenośny snapshot rekordów i diagnostyk todo2code;
4. aktywną, ratyfikowaną Konstytucję Organizacji;
5. ratyfikowany fundament intencji: wizję, misję i strategie organizacji.

Wynik ma kontrakt `subactor.project-change-analysis/v2`. Jest dry-runem:
nie tworzy ticketów, nie wykonuje URI, nie zmienia authority i nie interpretuje
propozycji jako akceptacji.

## Łańcuch pochodzenia intencji

Każda intencja projektowa musi dać się prześledzić w kolejności:

```text
Konstytucja → wizja → misja → strategia → cel projektu → ticket → EQL
```

Konstytucja pozostaje źródłem authority i invariants. Wizja opisuje pożądany
przyszły stan. Misja wyjaśnia cel istnienia i tożsamość organizacji. Strategia
przekłada misję na cele, priorytety, non-goals oraz jawny zakres projektów.

Kontrakt `subactor.organization-intent-foundation/v1` wiąże każdy poziom przez
niemutowalny `ref` i hash poprzednika. Daty ratyfikacji muszą zachować kolejność,
a strategia musi obejmować analizowany `project_id` albo zakres `*`. Sam fakt
pochodzenia nadal nie jest akceptacją Intent Contract, Intent Bindingiem ani
grantem wykonawczym.

### Bezpieczne spisanie fundamentu przez Foundera

Project Composer zawiera formularz „Spisz fundament intencji organizacji”.
Founder pozostaje autorem i jedynym ratyfikującym. Opcjonalny asystent może
przygotować szkic zaznaczonych pól z promptu Foundera, bieżącej treści i
aktywnej Konstytucji. Szkic jest propozycją do edycji w przeglądarce: nie jest
decyzją organizacji, nie zapisuje pliku i nie tworzy ticketu. Formularz
rozdziela trzy granice:

1. `POST /api/projects/intent-foundation/generate` wymaga `projects:read` oraz
   `llm:use`, przyjmuje jawny prompt i allow-listę pól, skanuje wejście oraz
   wynik pod kątem sekretów i zwraca kontrakt
   `subactor.organization-intent-foundation-draft/v1` z `review_required=true`;
2. `POST /api/projects/intent-foundation/preview` sprawdza aktywną Konstytucję,
   buduje następny dokument `organization-intent-foundation.vN.json`, wylicza
   hashe wizji, misji, strategii oraz całej fundacji i pokazuje semantyczny diff;
3. `POST /api/projects/intent-foundation/save` wymaga scope `routing:manage`,
   dokładnie tego samego `plan_hash`, niezmienionej poprzedniej rewizji i jej
   hasha, po czym tworzy nowy plik oraz przebudowuje Artifact Registry.

Generate i preview mają zero skutków ubocznych. Model może zwrócić tylko pola ze
ścisłego JSON Schema, a Control stosuje wyłącznie pola zaznaczone w żądaniu.
Niedostępny model, brak poprawnej Konstytucji, sekret, nieznane pole lub
niepoprawny identyfikator projektu kończy się fail-closed bez zmiany formularza.
Prompt nie trafia do audytu; zapisywany jest jego SHA-256 i lista pól. Każda
zmiana formularza po preview unieważnia przycisk zapisu oraz wymaga nowego
`plan_hash`.

Save nie nadpisuje istniejącej wersji, nie tworzy ticketów i nie uruchamia
deploymentu. Każda kolejna ratyfikacja trafia do nowego pliku `vN`; loader
wybiera najwyższą wersję i odrzuca ją fail-closed, gdy numer w dokumencie nie
zgadza się z nazwą pliku.

Hashe `vision.content_hash`, `mission.content_hash`, każdy
`strategy.content_hash` i `foundation_hash` są przeliczane przy podglądzie,
zapisie oraz odczycie. Poprawny format SHA-256 nie wystarcza, jeśli treść została
zmieniona. Preview i save skanują również dane wrażliwe, a loader i writer nie
akceptują rewizji będącej symlinkiem.

## Dzienny kierunek Foundera

Founder może opublikować opcjonalną, projektową dyrektywę na konkretny dzień:

```text
platform/config/governance/daily-directions/<YYYY-MM-DD>/<project-id>.vN.json
```

Kontrakt `subactor.founder-daily-direction/v1` zawiera fokus dnia, uporządkowane
priorytety, sukces dla każdego priorytetu, opcjonalny sprint, milestone’y i
non-goals. Każda rewizja wiąże hash fundamentu oraz strategii obejmującej
projekt, jest ratyfikowana przez Foundera i ma maksymalnie 36-godzinne okno
ważności. Najwyższa rewizja danego dnia wygrywa; wczorajsza dyrektywa nie jest
niejawnie przenoszona na dziś.

Dyrektywa zmienia uwagę i rekomendowaną kolejność pracy, ale nie authority.
Nie może naruszyć Konstytucji ani strategii, przyznać grantu, podać exact URI
lub ogłosić completion. Jej brak jest poprawnym stanem `not_set`, ponieważ jest
to opcjonalny instrument Foundera.

### Bezpieczne tworzenie i rewizja

Project Composer udostępnia formularz „Ustaw kierunek Foundera na dziś”. Zapis
jest rozdzielony na dwa jawne kroki:

1. `POST /api/projects/daily-direction/preview` waliduje fundament i strategię,
   buduje następną rewizję, pokazuje semantyczny diff oraz wylicza dokładny
   `plan_hash`; bilans efektów ubocznych wynosi zero;
2. `POST /api/projects/daily-direction/save` przyjmuje dokument z podglądu tylko
   razem z tym samym `plan_hash`, ponownie sprawdza `direction_hash`, bieżącą
   rewizję i fundament, po czym atomowo tworzy nowy plik oraz przebudowuje
   rejestr artefaktów.

Save ponownie wykonuje skan sekretów i sprawdza, czy provenance dokumentu
wskazuje dokładnie bieżącą poprzednią rewizję. Loader oraz writer odrzucają
plik rewizji będący symlinkiem.

Zapis wymaga scope `routing:manage` i włączonego Artifact Workspace. Preview
wymaga tylko `projects:read` i działa również w trybie read-only. Nie ma operacji
nadpisania: konflikt z nowszą rewizją zwraca błąd i wymaga nowego podglądu.

Docelowym źródłem jest montowany, wersjonowany workspace Platformy. Katalog
`CONTROL_DATA_DIR`, lokalny `.env` ani pamięć procesu nie przechowują decyzji.
Kontener czyta `/app/config`, lecz zapis trafia do odpowiadającego mu trwałego
`/workspace/platform/config`; oba wskazują ten sam hostowy katalog konfiguracji.

`direction_hash` jest ponownie wyliczany przy preview, save i każdym odczycie.
Dokument o poprawnym formacie hasha, ale zmienionej treści, jest odrzucany jako
`founder_daily_direction_hash_mismatch`.

## Tożsamość i zakres projektu

Wejście Project Composera ma opcjonalne `project_id`. Jeśli użytkownik go nie
poda, Control wyprowadza stabilny slug z nazwy. Istniejące tickety są wiązane z
projektem przede wszystkim przez `project:<id>`, `project-reconcile:<id>` albo
`purpose_context.scope.ref=project://subactor/<id>`. Dopasowanie po tytule jest
wyłącznie konserwatywnym fallbackiem.

Ticket z wykonaniem `running`, `claimed` lub `executing` jest aktywny. Ticket
`pending`, `ready`, `waiting_input` lub `queued` jest planowany. Stany terminalne
nie trafiają do operacyjnego snapshotu Delegation Managera.

## Przenośny snapshot todo2code

Trwałym źródłem nie jest lokalny cache `.intent/cache`. Pipeline publikuje
zarządzany plik w centralnej, wersjonowanej konfiguracji organizacji:

```text
platform/config/project-intent-evidence/<project-id>.v1.json
```

Plik ma schemat `subactor.project-intent-evidence/v1` i zawiera tylko ograniczoną
projekcję:

- `project_id`, `generated_at` i `graph_fingerprint`;
- wersję runtime i identyfikator runu;
- maksymalnie 500 kompaktowych rekordów intencji bez surowego kodu;
- maksymalnie 250 diagnostyk z cytowanymi `record_ids`;
- provenance i bezpieczne referencje źródeł.

Control przelicza liczniki diagnostyk zamiast ufać wartości zapisanej przez
producenta. Odczyt jest ograniczony rozmiarem, konwencjonalną nazwą pliku i
zwalidowanym `project_id`; ścieżka dostarczona przez klienta nie jest
akceptowana. Dzięki temu plik może być wersjonowany razem z projektem i
przenoszony między executorami razem z Platformą bez lokalnego `.env`.

Pełne grafy, raw excerpts i sekrety pozostają poza snapshotem. Snapshot jest
dowodem intencji, nie grantem, AQL ani decyzją Foundera.

## Bramka zgodności przed push i Pull Request

Platforma udostępnia jeden evaluator todo2code używany przez trzy wejścia:

- `npm run intent:check -- --repo <repo>` do ręcznego sprawdzenia zmiany;
- wersjonowany hook `platform/githooks/pre-push`, instalowany poleceniem
  `npm run intent:hooks:install`;
- kompozytową akcję `platform/actions/intent-conformance`, uruchamianą dla
  zdarzenia `pull_request` na dokładnym `base SHA` i `head SHA`.

Każdy przebieg porównuje tylko jedno repozytorium z jego własnym refem bazowym.
Obowiązkowa warstwa deterministyczna działa bez sieci i bez sekretów. Odrzuca
nieznany kontrakt todo2code, nowe diagnostyki `blocking` lub
`review_required` oraz materialny spadek implementation albo documented-code
coverage wynoszący co najmniej 1 punkt procentowy. Mniejszy spadek i nowy gap
są jawnie raportowane do review, ale same nie tworzą ticketu, nie zmieniają
authority i nie blokują historycznego długu bez dokładnej regresji.

Coverage jest miarą powiązania kodu z deklaracjami, dlatego materialny spadek
blokuje tylko wtedy, gdy diff zawiera plik źródłowy. Dla zmiany wyłącznie
konfiguracyjnej, dokumentacyjnej albo workflow ten sam spadek pozostaje jawnym
findingiem review. Nowe `blocking` i `review_required` blokują niezależnie od
rodzaju zmienionych plików.

Warstwa semantyczna działa tylko dla rzeczywistego diffu. Używa wyłącznie
`z-ai/glm-5.2` przez OpenRouter i trybu schema-bound todo2code. W trybie `auto`
brak sekretu powoduje jawne `skipped: credential_unavailable`, a kontrola
deterministyczna nadal obowiązuje. Tryb `required` fail-closed odrzuca zmianę,
jeżeli credentialu nie ma. Klucz jest wstrzykiwany z sejfu CI; model, pin
todo2code i reguły bramki są przenośnym kodem repozytorium, a nie lokalnym
`.env`.

Raport `subactor.intent-conformance-report/v1` wiąże projekt, bazę, commit,
listę zmienionych plików, trend, diagnostyczne delty, model oraz dokładny pin
todo2code. Jest dowodem kontroli jakości. Nie jest zgodą na merge, grantem ani
ratyfikacją intencji.

Ręczne polecenie analizuje bieżący filesystem razem z plikami
niecommitowanymi. Hook `pre-push` i workflow PR używają `--committed`: tworzą
krótkotrwały, izolowany worktree dokładnego `HEAD`, porównują go z bazą i po
zbudowaniu raportu usuwają worktree. Dzięki temu obce zmiany robocze nie
wchodzą do dowodu dotyczącego wysyłanego commitu, a CI i pre-push mają ten sam
zakres commit-bound.

## Zachowanie przy brakach

Jeżeli snapshot todo2code nie jest opublikowany, analiza ma stan wymagający
review i dodaje kandydaturę ticketu `INTENT-EVIDENCE-001`. Nie wykorzystuje
cache jako zastępczego faktu.

Jeżeli Konstytucja jest brakująca, nieważna, nieaktywna albo nieratyfikowana,
analiza ma stan `blocked` i dodaje blokującą kandydaturę ticketu:

```text
GOV-CONSTITUTION-001 — Napisz i ratyfikuj Konstytucję Organizacji
owner: founder
state: candidate_only
```

To jest kandydatura widoczna w dry-runie, nie ukryta mutacja Planfile. Control
produkcyjny nadal ładuje i waliduje Konstytucję fail-closed przy starcie.

Jeżeli nie ma zatwierdzonego fundamentu intencji, dry-run dodaje trzy zależne
od siebie kandydatury, których właścicielem jest Founder:

```text
GOV-VISION-001   — Spisz i ratyfikuj wizję organizacji
GOV-MISSION-001  — Spisz i ratyfikuj misję organizacji
GOV-STRATEGY-001 — Spisz i ratyfikuj strategię organizacji
```

Misja zależy od wizji, a strategia od misji. System nie generuje ich treści za
Foundera. Jeżeli fundament istnieje, ale żadna strategia nie obejmuje projektu,
powstaje kandydatura `GOV-PROJECT-STRATEGY-<PROJECT_ID>`.

## Wynik analizy

Wynik pokazuje:

- stan źródeł i ich referencje;
- aktywne i planowane tickety projektu;
- kandydatury duplikatów wobec nowego szkicu;
- rekordy todo2code pasujące do wskazania użytkownika, jawnie oznaczone jako
  dowody-kandydaci, a nie autorytatywne intencje;
- pełny status pochodzenia: Konstytucja, wizja, misja i wybrana strategia;
- niepokryte diagnostyki;
- ustalenia blokujące i wymagające review;
- tickety wymagane oraz rekomendowane jako `candidate_only`.

Diagnostyki todo2code mogą ugruntować zakres ticketu, ale nie zastępują Intent
Bindingu, authority, exact URI ani EQL. Materializacja pozostaje osobną,
niezaimplementowaną granicą wymagającą jawnej akceptacji i idempotentnego
kompilatora Planfile.

## Weryfikacja

Testy regresji obejmują:

- rozdzielenie ticketów aktywnych i planowanych;
- izolację ticketów innych projektów;
- wykrycie nakładającego się ticketu;
- pokrycie diagnostyki przez istniejącą kandydaturę;
- blokadę i ticket Foundera przy braku Konstytucji;
- sekwencyjne tickety Foundera przy braku wizji, misji i strategii;
- bezskutkowy preview fundamentu oraz zapis związany z dokładnym `plan_hash`;
- automatyczny wybór najwyższej poprawnej rewizji fundamentu;
- konflikt równoległej rewizji, tampering treści, sekret i symlink fundamentu;
- odrzucenie fundacji z odwróconą kolejnością ratyfikacji;
- blokadę projektu nieobjętego żadną ratyfikowaną strategią;
- wybór najnowszej rewizji dyrektywy dla bieżącego dnia w Europe/Warsaw;
- odrzucenie dyrektywy wygasłej albo związanej z inną strategią;
- rekomendację bounded ticketu, gdy dzisiejszy priorytet nie ma pokrycia;
- preview bez zapisu i zapis związany z dokładnym `plan_hash`;
- append-only rewizję oraz optymistyczny konflikt równoległej zmiany;
- odrzucenie treści zmienionej po wyliczeniu `direction_hash`;
- rekomendację publikacji evidence przy jego braku;
- odrzucenie traversal i niezgodnego `project_id`;
- zachowanie bilansu efektów ubocznych równego zero.
- odrzucenie nowego `blocking`, `review_required` i materialnego spadku pokrycia przez
  wspólny evaluator;
- jawne rozróżnienie semantycznego `passed`, `failed` i `skipped`;
- użycie GLM 5.2 bez przechowywania credentialu w repozytorium.

Pełny audyt spójności, wraz z wynikami todo2code i stanem ticketów
reconciliation, znajduje się w
[`intent-integrity-audit-2026-08-04.md`](./intent-integrity-audit-2026-08-04.md).
