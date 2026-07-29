---
{
  "schema": "subactor.doc/v1",
  "id": "docs.plans.refactoring-status-and-roadmap-2026-07-29",
  "version": 1,
  "status": "current",
  "updated": "2026-07-29"
}
---

# Status refaktoryzacji i plan dalszych prac

**Stan na:** 2026-07-29, Europe/Warsaw  
**Źródło diagnozy:** `project/analysis.toon.yaml` oraz świeże skany
`code2llm` podprojektów  
**Cel:** obniżać złożoność kodu bez zmiany kontraktów, zachować bramki
bezpieczeństwa i dostarczać każdą zmianę jako mały, testowalny commit.

## Podsumowanie wykonanych prac

Refaktoryzacje zostały wykonane i wypchnięte do gałęzi `main`:

| Repozytorium | Zakres | Commit(y) | Weryfikacja |
| --- | --- | --- | --- |
| `onedev-agent` | modele i storage zarządzania | `71f9371` | testy podprojektu |
| `test-agent` | modele E2E i CLI | `c5d03f6` | 48 testów |
| `doctor-agent` | walidacja, raporty, synchronizacja i naprawy | `543f71b`, `eb4ac26` | 99 testów |
| `repair-agent` | walidacja kontraktu, operacje naprawcze, patch i kontekst repozytorium | `ce5744d`, `135ac17`, `0e55f01` | 117 testów |
| `validator-agent` | walidacja komend | `a900160` | 89 testów |
| `github-com` | routing REST/GraphQL mock API | `4b2ee98` | 35 testów |
| `docs` oraz mirrory docs | rozdzielenie formatowania wyników operacji | `1927a9d`, `4193014`, `13462f0` | po 5 testów |
| `llm-code-benchmark` | wykrywanie findings, metryki agregowane i generowanie diffów file-edits | `35eb2aa`, `92afaa4`, `d6a7e8e` | 105 testów |

W `llm-code-benchmark` liczba hotspotów spadła z 12 do 9, a po refaktorze
`validator_benchmark.py` nie pozostały już zgłoszenia dla tych dwóch funkcji.
Nie zmieniono lokalnych, niezwiązanych plików `.env.example` ani istniejących
zmian w `core` i repozytorium dokumentacji.

## Co pozostało do wykonania

### P0 — ustalenie wiarygodnego baseline’u

1. Wygenerować ponownie `project/analysis.toon.yaml` po wszystkich
   refaktorach. Obecny plik główny nadal pokazuje część funkcji, które zostały
   już podzielone.
2. Oddzielić kod źródłowy od wygenerowanych bundli EQL/CDN. Bundli nie
   refaktoryzować ręcznie; trzeba wskazać ich generator i ewentualnie poprawić
   źródło.
3. Przed pracą w `core` zarejestrować zakres istniejących zmian użytkownika i
   uzgodnić osobny commit/gałąź. Nie mieszać ich z automatycznym refaktorem.

### P1 — następna kolejka bezpiecznych splitów

| Kolejność | Repozytorium / funkcja | CC | Proponowany zakres |
| --- | --- | ---: | --- |
| 1 | `llm-code-benchmark/live_benchmark` | 60 | rozdzielić orkiestrację runu, obsługę providerów i zapis wyników |
| 2 | `llm-code-benchmark/validate_unified_diff` | 30 | wydzielić parser nagłówków, walidację plików i reguły bezpieczeństwa |
| 3 | `llm-code-benchmark/_write_markdown` | 23 | wydzielić budowę sekcji, tabel i zapisu artefaktu |
| 4 | `llm-code-benchmark/parse_unified_diff` | 21 | rozdzielić tokenizację diffu od walidacji hunków |
| 5 | `repair-agent/_repair_operations` | 18 | rozdzielić planowanie, dry-run i wykonanie operacji |
| 6 | `validator-agent/update_release_metadata` | 30 | wydzielić odczyt, walidację i zapis metadanych |
| 7 | `validator-agent/_run_once` | 23 | rozdzielić przygotowanie procesu, wykonanie i raportowanie |
| 8 | `onedev-agent/snapshot` | 35 | wydzielić zbieranie danych, normalizację i zapis snapshotu |

Każdy punkt realizować osobnym małym commitem. Warunek akceptacji: brak zmiany
publicznego kontraktu, testy podprojektu przechodzą, `ruff`/lint przechodzi,
`git diff --check` przechodzi, a commit zawiera wyłącznie pliki danego zadania.

### P2 — porządki wtórne

- dokończyć hotspoty o CC 15–22 w `llm-code-benchmark`, `validator-agent` i
  `onedev-agent`;
- ponownie sprawdzić duplikaty po wygenerowaniu baseline’u;
- uzupełnić testy charakterystyczne dla wydzielonych helperów, jeżeli obecne
  testy sprawdzają tylko ścieżkę end-to-end;
- zaktualizować indeksy `analysis`, `evolution` i `CODEBASE_HEALTH` dopiero po
  potwierdzeniu finalnego zakresu zmian.

## Procedura realizacji

1. Odczytać aktualny hotspot ze świeżego skanu konkretnego podprojektu.
2. Sprawdzić status Git i odseparować lokalne zmiany użytkownika.
3. Wydzielić jedną odpowiedzialność bez zmiany sygnatur publicznych.
4. Uruchomić pełny zestaw testów podprojektu, lint i kontrolę diffu.
5. Zacommitować tylko pliki zadania i wypchnąć do `main`.
6. Odnotować commit, wynik testów i zmianę CC w tym dokumencie lub w kolejnym
   raporcie statusu.

## Kryterium zakończenia

Refaktoryzację tej kolejki można uznać za zakończoną, gdy:

- główny baseline jest świeży i rozróżnia źródło od artefaktów generowanych;
- wszystkie funkcje o CC > 15 mają zaakceptowany wyjątek albo plan splitu;
- nie ma niezaadresowanych duplikatów w kodzie źródłowym;
- każdy podprojekt ma zielony test suite po ostatnim commicie;
- zmiany użytkownika w `core`, dokumentacji i plikach środowiskowych są nadal
  odseparowane od commitów refaktoryzacyjnych.

