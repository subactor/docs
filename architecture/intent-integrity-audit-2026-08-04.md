---
{
  "schema": "subactor.doc/v1",
  "id": "docs.architecture.intent-integrity-audit-2026-08-04",
  "version": 6,
  "status": "current",
  "updated": "2026-08-04"
}
---

# Audyt integralności intencji i zmian projektów — 2026-08-04

## Zakres i metoda

Audyt porównał wskazania Foundera, Konstytucję, fundament intencji, Project
Composer, kontrakt kierunku dnia, aktywne i planowane tickety Planfile,
konfigurację reconciliation oraz fakty AST. todo2code uruchomiono
deterministycznie, bez LLM, komunikacji zewnętrznej i kosztu API.

Pierwszy szeroki przebieg całego workspace:

- runtime: `todo2code 0.5.0`;
- run: `20260804T101921Z-4efd259f`;
- fingerprint:
  `sha256:da12c313d4ff7d3090ba5f05a33ca622496b67f0712d3c13b1c9a516c1c0585e`;
- wynik: `succeeded`, 0 diagnostyk `blocking`;
- 397 `review_required`, 8 818 `warning`, 4 925 `info` w całym wielorepozytoryjnym
  workspace.

Wysokie liczby nie są miarą awarii Control. Dominują `IMPLEMENTED_NOT_DOCUMENTED`,
`IMPLEMENTED_NOT_PLANNED` i `UNLINKED_RECORD` dla historycznych repozytoriów,
TODO oraz changelogów. Dlatego szeroki wynik nie został opublikowany jako
projektowy evidence. Trwały snapshot musi być generowany z zakresu jednego
projektu, aby jego `project_id` i provenance nie wprowadzały w błąd.

Po naprawie dokumentacji wykonano zawężony przebieg dla repozytorium Core:

- run: `20260804T102518Z-720222cc`;
- fingerprint:
  `sha256:e2eebafb8e99434a016b9728019f0509ffb728c3c1d84696b5960d67e939c513`;
- wynik: `succeeded`, bez warnings i bez diagnostyk `blocking`;
- 177 `review_required`: 170 `AMBIGUOUS_REQUIREMENT` z heurystycznego rozbicia
  prozy dokumentacyjnej oraz 7 historycznych `CHANGELOG_WITHOUT_IMPLEMENTATION`;
- snapshot 500 rekordów i 250 bounded diagnostyk opublikowano jako
  `platform/config/project-intent-evidence/subactor-control.v1.json`.

System ma traktować te diagnostyki jako materiał do przeglądu. Nie są Intent
Bindingiem, authority ani automatyczną przesłanką do wykonania zmiany.

Po dodaniu autora fundamentu wykonano kolejny offline run na aktualnym
filesystemie Core:

- run: `20260804T104053Z-def367fd`;
- fingerprint:
  `sha256:b91d03bc31027a3117507b6cc0fc9f8b9d1cc42708214fbd2f523a6e3304d846`;
- wynik: `succeeded`, 0 diagnostyk `blocking`, 23 `review_required`,
  1 122 `warning` i 584 `info`;
- LLM był całkowicie wyłączony, a artefakty runu zapisano poza repozytorium.

Wzrost warningów wynika z tego, że analizator Core nie widzi osobnego
repozytorium Docs: nowe symbole AST są oznaczone jako
`IMPLEMENTED_NOT_PLANNED`, `IMPLEMENTED_NOT_DOCUMENTED` lub `UNLINKED_RECORD`,
choć dokument zarządzany istnieje w `docs/architecture`. To ujawnia rzeczywistą
lukę integracyjną todo2code dla wielorepozytoryjnego workspace, a nie dowód
braku implementacji. Dodano jawny wpis Core CHANGELOG, lecz pełne połączenie
cross-repo powinno pozostać osobnym ticketem zamiast być pozornie zamykane.

## Rozbieżności i decyzje

### 1. Formularz dziennego kierunku był opisany, ale niezaimplementowany — naprawione

Kontrakt i loader istniały, lecz Founder musiał ręcznie edytować JSON. Dodano
formularz w Project Composer, oddzielny preview/diff, dokładny `plan_hash`,
append-only save, atomowy zapis, przebudowę registry oraz optimistic conflict.

### 2. `direction_hash` nie dowodził integralności treści — naprawione

Loader sprawdzał tylko format `sha256:…`. Teraz ponownie wylicza hash z całego
dokumentu z wyłączeniem pola `direction_hash`. Zmiana treści po ratyfikacji jest
odrzucana fail-closed. Test regresyjny obejmuje tampering.

### 3. Dokumentacja deklarowała otwartą lukę po wdrożeniu kodu — naprawione

todo2code wskazał rozjazd „proposed” wobec istniejących symboli autora i testów.
Dokument architektury podniesiono do v5, a baza wiedzy otrzymała append-only v5.

### 3a. Dane wrażliwe i symlink poza workspace — naprawione

Dedykowany author nie może polegać wyłącznie na walidacji JSON. Preview skanuje
cały dokument pod kątem sekretów, a odczyt i zapis potwierdzają containment przez
`realpath`. Wklejony token oraz katalog daty będący symlinkiem poza workspace są
odrzucane przed utworzeniem pliku.

### 4. Fundament wizja → misja → strategia nie został jeszcze napisany — author naprawiony, treść nadal otwarta

Konstytucja `CONST-SUBACTOR` jest aktywna i ratyfikowana. Brakuje natomiast
zarządzanego `organization-intent-foundation.v1.json`. To świadomy brak decyzji
Foundera, nie błąd do automatycznego uzupełnienia. Project Composer pozostaje
zablokowany dla autorytatywnej intencji i pokazuje kandydatury:

- `GOV-VISION-001` — spisz i ratyfikuj wizję;
- `GOV-MISSION-001` — spisz i ratyfikuj misję, zależną od wizji;
- `GOV-STRATEGY-001` — spisz i ratyfikuj strategię, zależną od misji.

Formularz dziennego kierunku poprawnie odmawia preview do czasu istnienia
fundamentu oraz strategii obejmującej wybrany projekt.

Usunięto techniczną przeszkodę: Project Composer udostępnia teraz bezskutkowy
preview/diff i osobny exact-hash save całego fundamentu. Autor wylicza treściowe
hashe każdego poziomu, wiąże pochodzenie z Konstytucją i poprzednią rewizją,
skanuje sekrety, zapisuje atomowo i przebudowuje registry. Brakującą treść może
nadal podać i ratyfikować wyłącznie Founder.

### 4b. Brakowało kontrolowanego szkicu z promptu — naprawione bez rozszerzenia authority

Founder musiał dotąd przepisywać całą treść ręcznie, mimo że panel posiadał
centralną bramę LLM. Dodano generowanie wyłącznie jawnie zaznaczonych pól wizji,
misji i strategii. Control wymaga aktywnej Konstytucji, scope `projects:read`
oraz `llm:use`, skanuje prompt, kontekst i wynik pod kątem sekretów, a brama LLM
wymusza ścisły JSON Schema.

Wynik ma status `proposal` i `review_required=true`. Nie zapisuje fundamentu,
nie ratyfikuje treści, nie tworzy ticketów, nie nadaje authority i nie uruchamia
deploymentu. Niezaznaczone pola są ignorowane nawet wtedy, gdy model je zwróci.
Prompt nie jest przechowywany w audycie — pozostaje tylko hash i lista pól.
Edycja ręczna albo użycie szkicu unieważnia wcześniejszy preview; zapis jest
możliwy dopiero po ponownym diffie i wyliczeniu nowego exact `plan_hash`.

### 4a. Loader fundamentu czytał tylko `v1`, a hashe były syntaktyczne — naprawione

Runtime wybiera teraz najwyższą wersję `organization-intent-foundation.vN.json`
i sprawdza zgodność numeru dokumentu z nazwą pliku. `content_hash` wizji, misji
i strategii oraz `foundation_hash` są deterministycznie przeliczane. Zmiana
treści po ratyfikacji powoduje fail-closed zamiast cichego użycia artefaktu.

### 5. Anulowanie ticketu nie zmienia desired state projektu — zachowanie poprawne, lecz wymagające jasnego komunikatu

Live Planfile pokazuje `PLF-2848` jako `canceled`, natomiast późniejszy
`PLF-2868` jako `open/ready`. Manifest projektu nadal deklaruje
`docs-subactor-com → docs.subactor.com`, więc kontroler reconciliation ponownie
utworzył ticket dla nierozwiązanej rozbieżności. Anulowanie zatrzymuje konkretną
próbę; nie jest zmianą manifestu ani rezygnacją z celu.

Jeżeli Founder chce trwale zatrzymać publikację, potrzebna jest osobna,
audytowalna decyzja zmieniająca desired state projektu (np. suspend publication),
a nie usuwanie historii lub kolejnych ticketów naprawczych.

### 6. Szeroki snapshot todo2code może mieszać projekty — ograniczenie jawne

Exporter przyjmuje pojedynczy `project_id`; użycie grafu całego workspace
przypisałoby temu ID rekordy z innych repozytoriów. Snapshot projektowy należy
generować po zawężeniu root/patterns i dopiero wtedy publikować do
`platform/config/project-intent-evidence/<project-id>.v1.json`.

### 7. Snapshot connectora odrzucał poprawny binding deploymentu — naprawione

Live connector Plesk przyjmuje i ponownie waliduje `source_ref` oraz
`deployment_binding_ref`, `deployment_binding_hash` i
`deployment_binding_version`. Wersjonowany snapshot action surface nie zawierał
jednak tych czterech parametrów, więc pre-dispatch research odrzucał poprawny
dry-run jako `remediation_payload_invalid`. To była bezpośrednia techniczna
przyczyna kolejnych nieudanych cykli reconciliation po zgodzie Foundera.

Nie usunięto pól ochronnych z payloadu. Dodano deterministyczny refresher
snapshotu z live `urirun.bindings.v2`, uzupełniono 41 tras i test driftu.
Snapshot obejmuje teraz również `site/command/twin-sync` oraz
`site/query/twin-current`. Artifact Registry traktuje rejestry deployment
binding i twin jako rewizjonowane rejestry kontraktu v1, ponieważ ich pole
`version: 1` jest wersją schematu, a nie numerem edycji treści.

### 8. Testowy deployment docs w Digital Twin — zweryfikowane

Dodano przenośny profil `deployment-twin:docs-subactor-com`, zakotwiczony w
produkcyjnym bindingu `deployment:docs-subactor-com:production`, ale kierujący
wyłącznie do `docs-subactor-com.twin.test`. Profil wymusza brak sieci, zakaz
produkcyjnych credentiali i zapis tylko do izolowanego runtime root.

Live przebieg connectora:

- dry-run: 138 plików, `executed=false`, `mutation_attempted=false`, plan hash
  `9d7de3ec6547d197849fae6312328e15bef34a8bad39f9deaa0242eb4bbe303b`;
- apply w twin: release `rel_twin_9d7de3ec6547d197`, `verified=true`, transport
  `local-release-fs`, `source_matches_release=true`;
- entrypoint SHA-256:
  `0518fc22148b4ad4c5247b64dcd935efc122b7eb17b5c1e024fc860d57cedaf8`;
- niezależny `twin-current` potwierdził ten sam release i zgodność źródła.

Żadne wywołanie nie dotknęło Pleska ani `docs.subactor.com`.

Po tej naprawie ręczny reconciliation przeszedł bez błędu payloadu. Projekt
pozostaje `blocked` wyłącznie przez `mutation_gate_disabled`; rodzic to
`PLF-2868`, a bieżąca, jawna granica decyzyjna Foundera to `PLF-2872`.
Anulowanie starszego ticketu nie zmieniło desired state manifestu.

## Inwarianty po naprawie

- dzienny kierunek wynika wyłącznie z ratyfikowanego fundamentu i strategii;
- fundament jest append-only i może być napisany tylko przez jawny formularz
  Foundera z preview/diff przed zapisem;
- model może przygotować tylko ograniczony, jawnie wybrany szkic; nigdy nie
  ratyfikuje go ani nie zapisuje;
- każda edycja pól po preview unieważnia stary plan po stronie interfejsu;
- każdy poziom fundamentu i cały dokument mają weryfikowany hash treści;
- preview nie zapisuje pliku, nie tworzy ticketu i nie uruchamia deploymentu;
- save jest związany z dokładnym planem, treścią, target path i poprzednim hashem;
- preview odrzuca dane wrażliwe, a writer odrzuca symlink escape;
- rewizje są append-only, a konflikt wymaga ponownego preview;
- artifact registry musi zostać poprawnie przebudowany, inaczej nowy plik jest
  wycofywany;
- kierunek dnia nie nadaje authority, grantu ani prawa do wykonania URI;
- anulowanie ticketu zachowuje audyt i nie zmienia manifestu projektu.
- snapshot connectora musi pochodzić z live bindings i obejmować każdy parametr
  bezpieczeństwa emitowany przez builder payloadu;
- twin deploy nie ma sieci ani credentiali produkcyjnych i nie jest dowodem
  wykonania publikacji publicznej.

## Pozostała decyzja Foundera

Następnym krokiem organizacyjnym nie jest deployment. Founder powinien użyć
formularza fundamentu, spisać wizję, następnie misję, a następnie co najmniej
jedną strategię obejmującą projekty, obejrzeć dokładny diff i dopiero osobno
zatwierdzić zapis. Ten artefakt odblokuje dzienne priorytety jako pochodne
trwałej intencji organizacji.
