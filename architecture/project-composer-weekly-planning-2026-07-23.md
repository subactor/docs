---
{
  "schema": "subactor.doc/v1",
  "id": "docs.architecture.project-composer-weekly-planning-2026-07-23",
  "version": 1,
  "status": "current",
  "updated": "2026-07-23"
}
---

# Project Composer — opis projektu, graf pracy i symulacja tygodnia

## Cel

Project Composer zamienia opis człowieka w ograniczony szkic projektu,
kandydatury ticketów, propozycje przypisań oraz symulację pojemności ludzi i
automatów. Podgląd korzysta z bieżącego snapshotu delegacji i nie tworzy
ticketów Planfile, nie wykonuje URI Process, nie wysyła wiadomości i nie
zmienia authority.

Strona jest częścią zakładki `Projekty`:

```text
/?tab=projects&action=compose&week_start=YYYY-MM-DD&horizon=7
```

Opis projektu nie trafia do URL. URL zawiera wyłącznie bezpieczny stan widoku,
horyzont oraz ślad korelacji interakcji.

## Przepływ

```text
opis człowieka
  → normalizacja i kontrola sekretów
  → deterministyczny blueprint
  → opcjonalna propozycja LLM ze ścisłym JSON Schema
  → milestone’y i kandydatury ticketów
  → klasyfikacja domeny i wymaganych capabilities
  → aktualne reguły delegacji + AQL coverage
  → sugestia roli, gdy nie istnieje reguła
  → symulacja kolejki i pojemności
  → bottlenecki, braki i dry-run receipt
```

LLM może proponować nazwy procesów i zadania, lecz wynik jest ponownie
normalizowany. Pola wykonawcze, aktor, authority, URI, transport i sekrety nie
są przyjmowane z odpowiedzi modelu.

## Kontrakt wejściowy

`subactor.project-composer-request/v1` zawiera:

- nazwę, typ oraz opis projektu;
- początek i długość horyzontu;
- dzienną pojemność człowieka;
- dzienną pojemność automatu;
- jawny wybór użycia LLM.

Opis ma limit 12 000 znaków i odrzuca wartości wyglądające jak hasła, tokeny
albo klucze API. Nie przyjmuje URI ani poleceń wykonawczych.

## Kontrakt wyniku

`subactor.project-composer-preview/v1` zawiera:

- `draft_id` i deterministyczny `draft_hash`;
- cel projektu, milestone’y i kryteria sukcesu;
- kandydatury ticketów z tagami, estymatą i zależnościami;
- aktualne tickety uwzględnione w symulacji;
- wynik routingu i AQL dla każdej kandydatury;
- projekcję kolejki każdego aktora;
- przewidywane bottlenecki;
- braki implementacyjne;
- jawny bilans efektów ubocznych równy zero.

## Semantyka przypisania

Composer rozróżnia:

1. `automatic_rule` — obecna aktywna reguła delegacji wybrała aktora, a
   kontrakt aktora pokrywa wymagania.
2. `matched_rule` — reguła pasuje, ale wymaga ręcznej decyzji.
3. `proposed_role` — nie istnieje pasująca reguła; Composer jedynie symuluje
   aktora z właściwej roli i sprawdza jego AQL.
4. `unassigned` — nie istnieje aktywny aktor z wymaganym pokryciem.

`proposed_role` nie jest automatycznym przypisaniem i nie może być
materializowane bez jawnej polityki.

## Model tygodnia

Symulacja wykorzystuje:

- bieżące `estimated_busy_minutes`;
- projekcję backlogu pozostałego na początek horyzontu;
- odrębną dzienną pojemność ludzi i automatów;
- dni robocze dla ludzi i wszystkie dni dla automatów;
- sekwencyjne zależności zadań wewnątrz procesu;
- przyrost kolejki po każdym proponowanym przypisaniu.

`waiting_input` jest obowiązkiem uwagi, a nie czasem wykonawczym. Dlatego nie
zwiększa ETA, ale tworzy osobny bottleneck `attention_obligation`.

Estymata nowego zadania pozostaje heurystyką opartą o domenę oraz rozmiar
zadania. Przed materializacją musi zostać zaakceptowana albo zastąpiona
historycznym modelem estymacji.

## Stan implementacji

Zaimplementowano:

- stronę natural-language Project Composer;
- bezpieczny endpoint `POST /api/projects/composer/preview`;
- deterministyczny fallback;
- opcjonalną analizę przez istniejący LLM Gateway;
- normalizację odpowiedzi LLM;
- generowanie milestone’ów, tagów i zależności;
- routing przez aktualny Delegation Manager;
- AQL-aware dobór kandydatów;
- symulację pojemności i carry-in;
- listę aktualnych ticketów, kolejek i bottlenecków;
- zapis samego rekordu projektu Organization OS bez tworzenia ticketów;
- append-only zdarzenie `project_composer.previewed`.

## Braki blokujące materializację

### 1. Acceptance transition i Intent Binding

Intent Registry obsługuje `propose`, `revise`, `preview` i `diff`, lecz
celowo nie ma `accept` ani `execute`. Potrzebny jest transition wykonywany
przez uprawnionego principal-a oraz deterministyczny binding dokładnego
`intent_hash` do rewizji projektu.

### 2. Kompilator grafu do Planfile

Brakuje idempotentnego kompilatora:

```text
accepted intent + project graph
  → Planfile tickets
  → parent/source/dependency relations
  → Process Envelope
  → purpose context
  → materialization receipt
```

Ponowienie musi zwracać te same tickety albo semantic diff, nigdy duplikaty.

### 3. Kalendarze dostępności

Aktualne workloady nie zawierają urlopów, spotkań, godzin pracy ludzi ani
maintenance windows automatów. Composer przyjmuje pojemność ręcznie.

### 4. Historyczne estymaty

Brakuje modelu czasu opartego o wykonane tickety, domenę, capability,
wykonawcę oraz wariancję. Obecna projekcja jest jawnie oznaczona jako
heurystyczna.

### 5. Trwały graf zależności

Kandydatury mają zależności w szkicu, lecz produkcyjny adapter zapisu
cross-ticket dependency w Planfile nadal wymaga domknięcia.

## Następne etapy

1. Dodać acceptance transition i Intent Binding.
2. Zapisać wersjonowany Project Draft append-only.
3. Dodać podgląd semantic diff przed materializacją.
4. Zaimplementować idempotentny kompilator Planfile w trybie dry-run.
5. Dodać zatwierdzaną operację materializacji z receipt.
6. Podłączyć kalendarze i maintenance windows.
7. Kalibrować estymaty na podstawie ukończonych ticketów.

## Granice bezpieczeństwa

- LLM nie generuje authority, URI, transportu ani credentials.
- Symulacja nie przyznaje capability.
- Sugestia aktora nie jest delegacją.
- Rekord Organization OS nie jest ticketem wykonawczym.
- Materializacja pozostaje zablokowana do czasu wdrożenia Intent Binding.
- Każdy podgląd ma `correlation_id` w audycie i jawny bilans efektów ubocznych.

