---
{
  "schema": "subactor.doc/v1",
  "id": "docs.architecture.doql-digital-twin-situation-2026-07-22",
  "version": 1,
  "status": "current",
  "updated": "2026-07-22"
}
---

# DOQL/1 — Digital Twin Situation Query

## Co się stało z DOQL

DOQL nie został usunięty. W istniejącym pakiecie `oqlos/doql` pozostały parser,
model `DigitalTwinView`, bezpieczna projekcja `self` i deklaratywne pliki
`.doql.less`. Komenda `doql query` jest jednak nadal stubem. W Subactorze
Digital Twin został rozwinięty przede wszystkim jako subject-bound, read-only
widok pojedynczego aktora. Brakowało wykonawczej warstwy agregującej zbiór
obiektów do obrazu sytuacji.

Ten brak domyka `subactor.doql-situation-profile/v1`. W Subactorze skrót DOQL
oznacza teraz precyzyjnie **Digital Object Query Language**: deklaratywne,
ograniczone komponowanie sytuacji Digital Twin. Nie zastępuje DQL.

## Granice odpowiedzialności

| Warstwa | Odpowiada na pytanie | Czego nie robi |
|---|---|---|
| Intent Contract | czego chce człowiek lub maszyna? | nie przyznaje authority |
| DOQL | jaki jest zagregowany stan obiektów i jakie decyzje rozważyć? | nie diagnozuje przyczyny i nie wykonuje zmian |
| DQL | który inwariant jest naruszony? | nie naprawia |
| Strategy DSL | którą zatwierdzoną strategię wybrać? | nie nadaje uprawnień |
| AQL | kto może zdecydować lub wykonać? | nie opisuje operacji |
| OQL / URI Process | jakie kroki wykonać? | nie dowodzi rezultatu |
| EQL | czy oczekiwany rezultat został osiągnięty? | nie wykonuje naprawy |
| SODL / Planfile | co zaszło i jaki jest lifecycle pracy? | nie zastępuje stanu domenowego |

Kolejność kontrolna ma postać:

```text
obiekty domenowe → DOQL Situation Snapshot → DQL findings
                 → Strategy DSL → ticket/AQL → OQL/URI Process
                 → świeży snapshot → EQL receipt → SODL/Planfile
```

## Zaimplementowany przekrój

- Contracts zawiera JSON Schema i kanoniczny profil DOQL.
- Core zawiera deterministyczny evaluator operacji `count`, `count_where`,
  `sum`, `distinct_count` i `ratio`.
- Oceny sytuacji oraz wybór zadeklarowanych kandydatów decyzji korzystają z
  parsera wyrażeń TestQL, a nie z `eval` ani z kodu dostarczonego w profilu.
- Platform odczytuje wszystkie osiem `projekty/*/project.manifest.json` i
  uruchamia profil `organization.project-portfolio-situation`.
- Wynik DOQL jest przekazywany do osobnego profilu DQL, który sprawdza schema,
  read-only boundary oraz brak surowych obiektów.
- `npm run doql:check` jest częścią `test:meta`.

Aktualny read-back: 8 projektów, 8 domen, pełne pokrycie domen, unikalna
tożsamość domenowa, zero kandydatów naprawczych oraz 2/2 zielone inwarianty DQL.

## Kontrakt bezpieczeństwa

Profil wymusza `read_only=true`, `automatic_mutation_allowed=false` i
`include_raw_objects=false`. Każde źródło oraz cały snapshot mają limity liczby
obiektów. Raport przechowuje wyłącznie liczniki, hashe, metryki, oceny i
wcześniej zadeklarowane kandydatury decyzji.

Kandydat decyzji wskazuje ownera, `strategy_ref` i `eql_ref`, ale nie może
zawierać komendy, shella ani URI wykonawczego. Dalsza reakcja zawsze wymaga
ticketu, AQL i EQL. DOQL nie może sam wybrać sobie nowych możliwości.

## Następne etapy

1. Dodać typowane adaptery źródeł: tickety, procesy, DNS, WWW, konektory i SODL.
2. Wersjonować Situation Snapshot w registry i przechowywać freshness receipt.
3. Połączyć assessment z Strategy DSL bez omijania ticket-first preflight.
4. Dodać temporalne agregacje okien SODL i rozdzielić fakt, korelację oraz
   hipotezę przyczyny.
5. Wynieść wspólny parser/profil do `oqlos/doql`, zastępując stub `doql query`,
   dopiero po uzgodnieniu kompatybilności pakietu.

Nie należy tworzyć osobnego runtime do mutacji. DOQL jest read model/query
runtime w Core. Konektor URI jest potrzebny dopiero dla pozyskania zdalnego,
typowanego snapshotu; jego wynik pozostaje danymi wejściowymi, a nie authority.
