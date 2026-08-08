---
{
  "schema": "subactor.doc/v1",
  "id": "docs.plans.local-ci-control-plane-2026-08-08",
  "version": 1,
  "status": "current",
  "updated": "2026-08-08"
}
---

# Plan lokalnej płaszczyzny CI (act + OneDev) — 2026-08-08

## Cel

Doprowadzić do stanu, w którym **bramka jakości nie zależy od hostowanych
runnerów GitHuba** — ani od ich rozliczeń, ani od kolejek — przy zachowaniu
tego, co dziś działa zdalnie: check na PR, ten sam workflow, ten sam wynik.

Cel jest wąski i mierzalny: każde z 21 repozytoriów subactora, które ma
`.github/workflows`, dostaje status na PR wyprodukowany lokalnie.

## Stan na 2026-08-08

Trzy elementy istnieją i są sprawdzone. Nie są ze sobą spięte.

| Element | Co robi | Stan |
| --- | --- | --- |
| `onedev-agent` / `pr_verification.py` | odpytuje GitHub o otwarte PR-y, uruchamia `test_commands`, publikuje commit status `onedev/local-verify` | działa; `core` i `connectors` zielone |
| `github-com` (act 0.2.89, dind) | uruchamia `.github/workflows` lokalnie, wiele repo, zdarzenia `push`/`pull_request`/`workflow_dispatch` | obraz zbudowany, oba testy przeszły |
| `scripts/verify` + `.githooks/pre-push` | bramka lokalna wołana ręcznie i przed pushem | jest w `core`, `connectors`, `diagit` |

### Co zostało udowodnione pomiarem

Nie założone — uruchomione:

| Workflow | Lokalnie przez act | Hostowany runner |
| --- | --- | --- |
| `diagit / verify.yml` (samowystarczalny) | ✅ ruff, mypy 19 plików, 21 testów, wheel | 23 s |
| `core / intent-conformance.yml` (prywatna akcja + pinowany todo2code) | ✅ **3m08s** | 2m36s |

Przepis, który zadziałał — trzy warunki, każdy konieczny:

1. `GITHUB_TOKEN` z `gh auth token` w pliku `ACT_SECRET_FILE`. Deploy key
   `PLATFORM_ACTION_DEPLOY_KEY` **nie jest potrzebny**: `actions/checkout`
   z pustym `ssh-key` schodzi do tokena, a token czyta prywatne repo.
2. **Prawdziwy payload PR-a** z `gh api /repos/O/R/pulls/N`. Wysyłany
   `config/events/pull_request.json` ma zaślepki (`1111…`, `2222…`,
   `local/repository`) i daje `upload-pack: not our ref`.
3. Sekrety specyficzne dla workflow (tu `OPENROUTER_API_KEY`).

Wywołanie, bez modyfikacji `.env` orkiestratora:

```sh
REPOSITORIES_ROOT=/home/tom/github/subactor \
docker compose run --rm \
  -e REPOSITORIES=core \
  -e ACT_EVENTS=pull_request \
  -e ACT_WORKFLOW=intent-conformance.yml \
  -e ACT_SECRET_FILE=/config/secrets.local.env \
  -e ACT_EVENT_FILE_PULL_REQUEST=/config/events/pull_request.local.json \
  runner
```

### Luki wobec stanu kodu

| Luka | Objaw | Gdzie |
| --- | --- | --- |
| Szablon zdarzenia to zaślepka | `fatal: upload-pack: not our ref 1111…` | `github-com/config/events/pull_request.json` |
| Weryfikacja uruchamia komendy, nie workflowy | pokrycie 2 repo zamiast 21 | `onedev-agent/src/onedev_agent/pr_verification.py` |
| `dependency_links` ustalane ręcznie | dla `core` siedem pozycji, w tym `../projekty` | `onedev-agent/config/repositories.toml` |
| `pr-executor` bez sieci | `act` potrzebuje socketu Dockera i pobierania obrazów | `onedev-agent/compose.yaml` |
| `core` niebudowalny samodzielnie | 81 → 73 → 5 → 0 awarii w miarę dokładania rodzeństwa | `core` (runtime + 3 pakiety + `platform`/`contracts`/`projekty`) |
| Dwie płaszczyzny sterowania | `diagit` i `onedev-agent` mają nakładającą się władzę nad tym samym zbiorem repo | — |

## Slice P0 — spiąć to, co już działa

Najmniejsza zmiana dająca skokowy wzrost pokrycia. `pr_verification.py`
**już** pobiera obiekt PR-a, żeby wykryć nowy `head_sha` — ma więc w ręku
dokładnie dane, z których buduje się payload.

- [ ] P0.1 Zapisywać payload zdarzenia z pobranego PR-a w kształcie
      `{"action","number","pull_request","repository","sender"}` zamiast
      używać szablonu z zaślepkami.
- [ ] P0.2 Dodać w `repositories.toml` tryb wykonania `workflow` obok
      `test_commands`, wołający runner `github-com` z payloadem z P0.1.
- [ ] P0.3 Dać `pr-executor` socket Dockera i sieć; udokumentować, że
      izolacja sieciowa została świadomie wymieniona na możliwość
      uruchamiania workflowów.
- [ ] P0.4 Objąć trybem `workflow` `diagit` i `core`, czyli oba przypadki
      brzegowe: samowystarczalny i z prywatną zależnością.

Kryterium wyjścia: PR w `diagit` i w `core` dostaje status z przebiegu
lokalnego, bez zużycia minut Actions.

## Slice P1 — pokrycie i odkrywanie zależności

- [ ] P1.1 Rozszerzyć tryb `workflow` na pozostałe repozytoria z
      `.github/workflows` (21 łącznie; 18 uruchamia testy).
- [ ] P1.2 Zautomatyzować wykrywanie sprzęgnięć zamiast ręcznych
      `dependency_links`. Wzorzec istnieje:
      `observability/scripts/run-ci-tests.mjs` wylicza domknięcie importów
      każdego pliku testowego i pomija tylko te, które faktycznie potrzebują
      nierozwiązywalnego pakietu — zamiast twardej listy, „która po cichu
      zgniła".
- [ ] P1.3 Wprowadzić `npm run verify` jako jedyną komendę bramki w każdym
      repo. Wtedy konfiguracja zwija się do jednej pozycji bez
      `dependency_links`, bo verify sam deklaruje i dowiązuje rodzeństwo.

## Slice P2 — ład

- [ ] P2.1 Rozstrzygnąć podział władzy między `diagit` a `onedev-agent`:
      który jest autorytatywny, gdy oba widzą ten sam stan inaczej.
- [ ] P2.2 Dodać `diagit` do allowlisty OneDev (dziś 375 repo, `diagit`
      poza nią), żeby narzędzie rządzące flotą samo podlegało jej maszynerii.
- [ ] P2.3 Opisać retencję `audit.jsonl`. Rotacja z 2026-08-08 zresetowała
      rozmiar (251 MB → 0), nie przyczynę: plik rośnie ~15 MB/dobę, dziś
      dominuje `vault.lease` (~3,4/min po 2,4 KB).

## Ryzyka i granice

**`act` jest przybliżeniem hostowanego runnera.** Artefakty, cache i
uprawnienia `GITHUB_TOKEN` zachowują się inaczej. Do bramki testowej
wystarcza — oba przebiegi to potwierdziły — ale nie należy obiecywać
wierności dla workflowów zależnych od usług GitHuba.

**Token o szerokim zakresie zastępuje deploy key.** To upraszcza
uruchomienie i **poszerza** uprawnienia względem klucza wystawionego na jedno
repo. Świadomy wybór, nie efekt uboczny.

**Rozliczenia Actions to epizod, nie stan.** Blokada trwała od ~05:42Z do
~10:16Z 2026-08-08 i została zdjęta. Ten plan nie zakłada, że Actions nie
działa — zakłada, że **nie chcemy być od nich zależni**, co jest inną i
trwalszą przesłanką.

**Brak branch protection na planie free.** Czerwony status nie zablokuje
merge'a. Bramka lokalna i hook `pre-push` są jedynym realnym egzekwowaniem,
dopóki to się nie zmieni.
