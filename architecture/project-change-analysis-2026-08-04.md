---
{
  "schema": "subactor.doc/v1",
  "id": "docs.architecture.project-change-analysis-2026-08-04",
  "version": 5,
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

Pełny audyt spójności, wraz z wynikami todo2code i stanem ticketów
reconciliation, znajduje się w
[`intent-integrity-audit-2026-08-04.md`](./intent-integrity-audit-2026-08-04.md).
