# Architektura Subactor Organization OS

## Warstwy

### Organization Core

Przechowuje kanoniczne rekordy organizacji. Udostępnia REST CRUD, wyszukiwanie,
dashboard, miękką archiwizację, seeding developerski i wewnętrzne operacje OQL.

### Control Panel

Obsługuje tokeny, AQL/OQL, zatwierdzenia, audyt oraz proxy do Organization Core.
Przeglądarka nie otrzymuje sekretów usług.

### AQL

Klasyfikuje sytuację na podstawie typowanych wejść. Pierwsza reguła według priorytetu
wybiera wariant, odbiorców, kroki i reason code.

### OQL

Jest czytelnym planem do zatwierdzenia. Plan nie wykonuje się przed przejściem do
statusu `approved`.

### Bridge

Wykonuje tylko operacje z katalogu. Operacje `org.*` trafiają do Organization Core,
a komunikacyjne do Pleska, SMTP, Slacka, Teams, webhooków albo kalendarza.

### LLM Gateway

Jedyny komponent posiadający klucz OpenRouter. LLM tworzy typowany JSON albo projekt
wiadomości. Nie zatwierdza ani nie wykonuje planów.

## Granice domen

```text
People Operations      ludzie, zespoły, onboarding
Customer Operations    klienci, kontakty, relacje
Communication          rozmowy, wiadomości, zobowiązania
Contract Operations    umowy i wersje
Project Operations     projekty, zadania, blokery
Strategy Operations    decyzje i realizacja strategii
```

Wszystkie domeny współdzielą identyfikatory, tokeny, audyt i mechanizm AQL/OQL,
ale zachowują odrębne zakresy uprawnień.

## Project Business Layer (v2.1)

`project-importer` jest osobną usługą z własnym tokenem serwisowym. Nie zapisuje danych
bezpośrednio do plików Organization Core — korzysta z jego API.

```text
hr-control
  ├── project-importer
  │     ├── website importer
  │     ├── directory importer
  │     ├── git importer
  │     ├── Markdown normalizer
  │     ├── deterministic blueprint
  │     ├── optional LLM analysis
  │     └── import TestQL
  ├── AQL/OQL plan registry
  └── Organization Core
        └── project workspace
```

Bridge wykonuje operacje `org.*` oraz `testql.project.run`. Wszystkie działania
zmieniające organizację pozostają za planem OQL i approval.
