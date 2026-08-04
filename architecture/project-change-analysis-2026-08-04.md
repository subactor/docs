---
{
  "schema": "subactor.doc/v1",
  "id": "docs.architecture.project-change-analysis-2026-08-04",
  "version": 2,
  "status": "current",
  "updated": "2026-08-04"
}
---

# Analiza zmian projektu — user guidance, Planfile, todo2code i Konstytucja

## Cel

Project Composer przed materializacją zmian porównuje cztery niezależne źródła:

1. aktualne wskazanie użytkownika i wynikający z niego deterministyczny szkic;
2. aktywne oraz planowane tickety Planfile przypisane do `project_id`;
3. przenośny snapshot rekordów i diagnostyk todo2code;
4. aktywną, ratyfikowaną Konstytucję Organizacji.

Wynik ma kontrakt `subactor.project-change-analysis/v1`. Jest dry-runem:
nie tworzy ticketów, nie wykonuje URI, nie zmienia authority i nie interpretuje
propozycji jako akceptacji.

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

## Wynik analizy

Wynik pokazuje:

- stan źródeł i ich referencje;
- aktywne i planowane tickety projektu;
- kandydatury duplikatów wobec nowego szkicu;
- rekordy todo2code pasujące do wskazania użytkownika;
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
- rekomendację publikacji evidence przy jego braku;
- odrzucenie traversal i niezgodnego `project_id`;
- zachowanie bilansu efektów ubocznych równego zero.
