---
{
  "schema": "subactor.doc/v1",
  "id": "docs.operations.uri-twin-plesk-deploy-runbook",
  "version": 4,
  "status": "current",
  "updated": "2026-07-29"
}
---

# Runbook: deploy / rollback uri-twin Plesk observe

## Cel
Włączyć `plesk://…/query/…` twin-facts w żywym `urirun-node` + Control reality-check
bez zmian HITL / mutate.

## Deploy
1. Kod connectora jest bind-mount: `~/github/urirun-connectors` → `/connector-repos`.
2. Control `src` jest bind-mount: `core/services/control/src` → `/app/services/control/src`.
3. `urirun-connector-subactor-twin-map` ładuje domyślnie
   `https://github.com/uri-twin/uri-twin-plesk.git` (`main`). Opcjonalne
   override'y operacyjne: `URI_TWIN_PLESK_REPOSITORY`, `URI_TWIN_PLESK_REF`,
   `URI_TWIN_PLESK_PATH`, `URI_TWIN_CACHE_DIR`, `URI_TWIN_OFFLINE`.
4. Kanoniczne wdrożenie całego lokalnego stosu składa bazowy Compose, bezpieczny
   Control i bramę connector LAN. Polecenie wykonuje preflight, buduje obrazy,
   czeka na gotowość, sprawdza URIrun i mTLS oraz uruchamia wyłącznie podgląd
   kontrolera:

```bash
cd ~/github/subactor/platform
npm run stack:deploy
```

Pełny restart bez usuwania wolumenów: `npm run stack:restart`. Sam preflight:
`npm run stack:check`; stan żywego stosu: `npm run stack:status`. Opcja
`--skip-controller` pomija read-only preview. Skrypt nigdy nie dodaje overlayu
execute-once ani nie uruchamia kontrolera z `--execute`.

5. Smoke (wymaga `SUBACTOR_ADMIN_TOKEN` z `.env`):

```bash
export SUBACTOR_CONTROL_URL=http://127.0.0.1:8091
# routes
subactor get /api/connector-runtime | jq -r '.routes[].uri' | rg 'plesk://host/(site/query/docroot|subscription/query/snapshot|dns/query/authority)'
# Git-backed dynamic map
subactor get /api/connector-runtime | jq -r '.routes[].uri' | rg 'twin://plesk/map/query/(snapshot|resolve|refresh)'
# URI Process twin-fact
subactor uri 'plesk://host/site/query/docroot' '{"domain":"docs.subactor.com","main_domain":"subactor.com"}'
# reality-check: bridge projection
subactor get '/api/projects/reality-check?domain=docs.subactor.com'
# reality-check: URI Process preferred
subactor get '/api/projects/reality-check?domain=docs.subactor.com&via=uri'
```

## Akceptacja
- Trzy URI widoczne w `route_count` / liście routes.
- `via=uri` → `sources.plesk_docroot_twin.source == "uri_process"` i `schema == subactor.twin-fact/v1`.
- Bez `via` → `source == "bridge_http"` (projekcja), fail-open.
- `twin://plesk/map/query/snapshot` → `baseline.loaded_from == "git"`, pełny
  commit, digest i `stale == false`; fallback cache/embedded musi mieć
  `stale == true`.
- `observe-topology` zwraca query plan dependency-first. `publish-*` z
  docroot fact `decision=refuse` zwraca pusty plan i
  `execution_policy=precondition-blocked`.
- `human_boundary` / granty nietknięte.

## Live smoke mapy (2026-07-29)

- `urirun-node`: 644 trasy, w tym trzy `twin://plesk/map/query/*`.
- Git baseline: commit `069008bff9c08366d2e3e156c4c9b5b937eaa37e`,
  baseline v3, `loaded_from=git`, `stale=false`.
- Plesk: 19 subskrypcji; subscription/docroot/DNS facts `fresh`; katalog
  rozszerzeń dostępny; dynamiczna mapa zawierała 22 zasoby i zero discoveries.
- `observe-topology`: resolved, dwa kroki query.
- `publish-site-with-managed-dns`: zatrzymany przed command przez
  `fact_refused: plesk.site.docroot` (zero kroków wykonawczych).

## Live smoke (2026-07-25, Subactor)

| Domain | via=uri authority | decision | observed |
| --- | --- | --- | --- |
| docs.subactor.com | observed | accept | /docs.subactor.com |
| logo.subactor.com | observed | accept | /logo.subactor.com |
| subactor.com | observed | accept | /httpdocs |
| www.subactor.com | rule (domain missing in Plesk) | accept | — |
| identity.subactor.com | rule (DNS + domain missing) | accept | — |

KPI `via=uri`: **3/5 observed (60%)** — non-observed cases are missing Plesk objects / DNS, not twin regressions.
DNS authority twin: Cloudflare NS consensus `fresh` for `subactor.com`.

## Rollback

Jeżeli nowe kontenery nie osiągną gotowości, `stack:deploy` podejmuje próbę
odtworzenia poprzednich lokalnych obrazów i zachowuje wolumeny, po czym wypisuje
diagnostykę. Bind-mounty nadal wskazują bieżący worktree, więc pełny rollback
źródeł wymaga wskazania poprzedniego commita:

```bash
# przywróć poprzedni commit w urirun-connector-plesk + core, potem:
npm run stack:restart -- --no-build
```
HTTP `/plesk/docroot` i stare reality-check bez twinów wracają automatycznie po
cofnięciu kodu; mutacje nie zależą od query twin.

Awaria GitHub nie wymaga rollbacku: loader użyje zwalidowanego last-known-good,
a następnie embedded baseline i oznaczy go `stale`. Wymuszenie kontrolowanego
offline: `URI_TWIN_OFFLINE=1`; nie traktować takiej mapy jako dowodu aktualności
repozytorium.

## KPI (ręczny odczyt)
Z reality-check reportów: udział `authority=observed` vs `rule` / `fact_quality=estimated`.
