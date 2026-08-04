---
{
  "schema": "subactor.doc/v1",
  "id": "docs.architecture.intent-integrity-audit-2026-08-04",
  "version": 3,
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

### 4. Fundament wizja → misja → strategia nie został jeszcze napisany — otwarte

Konstytucja `CONST-SUBACTOR` jest aktywna i ratyfikowana. Brakuje natomiast
zarządzanego `organization-intent-foundation.v1.json`. To świadomy brak decyzji
Foundera, nie błąd do automatycznego uzupełnienia. Project Composer pozostaje
zablokowany dla autorytatywnej intencji i pokazuje kandydatury:

- `GOV-VISION-001` — spisz i ratyfikuj wizję;
- `GOV-MISSION-001` — spisz i ratyfikuj misję, zależną od wizji;
- `GOV-STRATEGY-001` — spisz i ratyfikuj strategię, zależną od misji.

Formularz dziennego kierunku poprawnie odmawia preview do czasu istnienia
fundamentu oraz strategii obejmującej wybrany projekt.

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

## Inwarianty po naprawie

- dzienny kierunek wynika wyłącznie z ratyfikowanego fundamentu i strategii;
- preview nie zapisuje pliku, nie tworzy ticketu i nie uruchamia deploymentu;
- save jest związany z dokładnym planem, treścią, target path i poprzednim hashem;
- preview odrzuca dane wrażliwe, a writer odrzuca symlink escape;
- rewizje są append-only, a konflikt wymaga ponownego preview;
- artifact registry musi zostać poprawnie przebudowany, inaczej nowy plik jest
  wycofywany;
- kierunek dnia nie nadaje authority, grantu ani prawa do wykonania URI;
- anulowanie ticketu zachowuje audyt i nie zmienia manifestu projektu.

## Pozostała decyzja Foundera

Następnym krokiem organizacyjnym nie jest deployment. Founder powinien najpierw
spisać wizję, następnie misję, a następnie co najmniej jedną strategię obejmującą
projekty. Dopiero ten artefakt odblokuje dzienne priorytety jako pochodne trwałej
intencji organizacji.
