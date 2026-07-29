---
{
  "schema": "subactor.doc/v1",
  "id": "docs.platform.codebase.health",
  "version": 2,
  "status": "current",
  "updated": "2026-07-29"
}
---

# Stan kodu i plan refaktoru (code2llm)

**Data indeksu:** 2026-07-17  
**Źródła:** `project/analysis.toon.yaml`, `project/map.toon.yaml`, `project/evolution.toon.yaml`, `project/planfile-tickets.yaml`

## Źródło prawdy (monorepo workspace)

| Ścieżka | Rola |
|---------|------|
| `core/`, `agents/`, `connectors/`, `runtime/`, `contracts/`, … | Kanoniczne repozytoria komponentów (osobne `.git`) |
| `platform/components/<name>/` | **Symlinki** do kanonicznych repo (`../../<name>`); `platform/.gitignore` ignoruje `/components/` |
| `platform/dependencies.lock.json` | Przypięcie każdego komponentu do commita — **jedyne** źródło wersji dla deployu |
| `platform/` | Docker Compose, config, skrypty, dokumentacja operacyjna |

**Nie ma submodułów.** `verify-dependencies-lock.mjs` aktywnie ich zakazuje (`.gitmodules is forbidden`, gitlinks odrzucane). Assembly to `external-git-checkouts`: `dependencies:sync` odtwarza drzewo z locka, a dla symlinkowanego rodzeństwa tylko asertuje zgodność HEAD.

**Zasada edycji:** komponent edytuje się **raz**, w kanonicznym repo — `platform/components/*` to ten sam i-node. Osobna kopia w `components/` jest defektem (patrz „Dług strukturalny").

**Zasada wersjonowania:** po wypchnięciu komponentu przypnij nową wersję przez `npm run dependencies:pin`. Lock nigdy nie może wskazywać commita spoza `origin/main` — `dependencies:sync` pobiera commit z remote, więc lokalny pin daje workspace nieodtwarzalny dla innych.

Skan code2llm podąża za symlinkami i widzi drzewo komponentu dwa razy — metryki CC/duplikacji są **sztucznie podwojone**. Przy triażu ticketów planfile bierz ścieżki bez prefiksu `platform/components/`.

## Metryki (indeks 2026-07-17)

| Metryka | Skan surowy | Po deduplikacji lustra |
|---------|-------------|------------------------|
| LOC | ~52 382 | ~26 645 |
| Funkcje | 5 063 | — |
| Moduły | 409–427 | ~238 kanonicznych |
| CC̄ | 3.9 | — |
| high-CC (≥15) | 226+ | połowa to mirror |
| Cykle importów | 0 | OK |

**Języki:** JavaScript/MJS dominuje, potem PHP (portal, site-generator), shell, mało Pythona.

## Hotspoty (kolejność refaktoru)

Pomiar LOC z 2026-07-29; wartości CC pochodzą z indeksu 2026-07-17 i **nie zostały
przemierzone** — traktuj je jako historyczne.

| Priorytet | Plik | LOC 07-17 | LOC 07-29 | Problem |
|----------:|------|----------:|----------:|---------|
| P0 | `core/.../control/src/server.mjs` | ~954 | **3069** | już nie router (handler ~220 L); shared kernel: 139 stałych env, 82 importy, 117 funkcji, `routeDeps()` = worek 130 kluczy, 6 schedulerów `setInterval`. Jedyny moduł control **bez testu** |
| P0 | `core/.../control/public/app.js` | ~2485 | **5499** | 335 funkcji top-level, 3 instrukcje import/export — monolit ES bez podziału; cache-busting ręczny (`?v=…`) |
| P1 | `connectors/.../bridge/src/server.mjs` | ~1313 | **2052** | wydzielenie zatrzymane w połowie: `plesk-*.mjs` istnieją, ale `pleskFetch`/`pleskCreate`/`pleskSiteSync`/`pleskPublishSite` dalej w `server.mjs`; `fetchJson` (165 L) prawdopodobnie duplikuje `runtime/service-client` |
| P1 | `core/.../control/src/delegation-coverage.mjs` | — | 141 | `actorCoversRequirements` bez **żadnego** testu; znany defekt: wildcard `**` nie liczy się jako pokrycie (SYSTEM_STATE_2026-07-24) |
| P2 | `contractor-portal/.../index.php` | ~1124 | 1219 | — |

Kierunek jest przeciwny do celu: trzy główne hotspoty urosły 1,6–2,2× w 12 dni,
mimo że zaplanowany refaktor został wykonany. Rozbicie na moduły działa
(control: 110 modułów `src/` + 29 `routes/`, 111 plików testowych), ale nie
nadąża za tempem dokładania funkcji.

## Dług strukturalny

Groźniejszy od LOC, bo dotyczy bramki deployu i źródła prawdy uprawnień.

| Pozycja | Stan 2026-07-29 |
|---|---|
| `platform/components/autonomy-lab` | **naprawione** — był osobnym, nieaktualnym klonem (`7149555` vs lock `c120264`), `live-observer.mjs` pusty zamiast 279 L; zamieniony na symlink |
| Lock vs rzeczywistość | `connectors`, `core`, `observability`, `runtime` rozjechane; `dependencies:verify` czerwone |
| `core`, `runtime` | HEAD **niewypchnięty** — do czasu pusha locka nie wolno przypiąć |
| `contractor-portal/vendor/contracts/` | zwendorowana kopia repo `contracts/`, już rozjechana (brak 8 katalogów aktorów, m.in. `safety-operator`, `security`, `access`); AQL jest źródłem prawdy dla uprawnień |
| `projekty/contracts-subactor-com/` | zapomniany fork `contractor-portal/`, starszy o poprawki a11y i `chat_linkify_http_urls`; wewnątrz `app/index.php` = `app/public/index.php` bajt w bajt |
| `TODO.md` (785 pozycji, prefact) | nieużyteczny jako backlog: mirrory i `vendor/` liczone 2–3×, treść kosmetyczna; realny dług się tam nie pojawia |

## Cele ewolucji (code2llm)

- CC̄: 3.9 → ≤2.8  
- max-CC: 280 → ≤20  
- god-modules: 27 → 0  
- high-CC (≥15): 226 → ≤113  

## Faza w toku (2026-07-29)

| Krok | Status |
|------|--------|
| Dokumentacja health + plan | **done** |
| Split `delegation-manager` → config / coverage / decision | **done** (1038 → 360 L) |
| Extract `access-registry`, `integration-records`, `delegation-summary` | **done** |
| Routing HTTP control server (handlery per domena) | **done** — `src/routes/`, 29 modułów, 5767 L, `dispatch.mjs` |
| `platform/components/autonomy-lab` → symlink | **done** (2026-07-29) |
| `dependencies:pin` — odświeżanie locka z guardami | **done** (2026-07-29) |
| Odświeżenie locka dla 4 rozjechanych komponentów | **blocked** — wymaga pusha `core` i `runtime` |
| Rozbicie `routeDeps` (130 kluczy) na deps per-domena | **next** |
| Wydzielenie 6 schedulerów z `server.mjs` do `jobs/` | next |
| Dokończenie `plesk-*` w bridge, dedup `fetchJson` | planned |
| Modularizacja `app.js` (start: domena delegowania) | planned |
| Testy charakteryzujące `delegation-coverage`, potem naprawa `**` | planned |
| Usunięcie `vendor/contracts` i forka `projekty/contracts-subactor-com` | planned |

Testy platformy `test:meta`: 143/143 zielone (2026-07-29).

## Jak odświeżyć indeks

```bash
# z roota workspace; wyklucz node_modules i venv
code2llm ./ -f all --toon-yaml -o project --no-png \
  --exclude node_modules .venv platform/node_modules
```

Ticket’y z code2llm: `project/planfile-tickets.yaml` (495 sygnałów; ~połowa to mirror).  
Aktywna kolejka Koru/planfile w workspace może być pusta — backlog refaktoru trzymamy w docs + PR.

## Powiązane

- [ORGANIZATION_OS_ARCHITECTURE.md](./ORGANIZATION_OS_ARCHITECTURE.md)  
- [platform/docs/refactoring-summary-and-next-steps-2026-07-16.md](../../platform/docs/refactoring-summary-and-next-steps-2026-07-16.md)  
- [platform/docs/DELEGATION_MANAGER.md](../../platform/docs/DELEGATION_MANAGER.md)  
- Indeksy: `project/analysis.toon.yaml`, `project/map.toon.yaml`
