---
{
  "schema": "subactor.doc/v1",
  "id": "docs.operations.uri-twin-plesk-deploy-runbook",
  "version": 2,
  "status": "current",
  "updated": "2026-07-25"
}
---

# Runbook: deploy / rollback uri-twin Plesk observe

## Cel
Włączyć `plesk://…/query/…` twin-facts w żywym `urirun-node` + Control reality-check
bez zmian HITL / mutate.

## Deploy
1. Kod connectora jest bind-mount: `~/github/urirun-connectors` → `/connector-repos`.
2. Control `src` jest bind-mount: `core/services/control/src` → `/app/services/control/src`.
3. Rebuild registry + reload Node:

```bash
cd ~/github/subactor/platform
docker compose restart urirun-node hr-control hr-bridge
```

4. Smoke (wymaga `SUBACTOR_ADMIN_TOKEN` z `.env`):

```bash
export SUBACTOR_CONTROL_URL=http://127.0.0.1:8091
# routes
subactor get /api/connector-runtime | jq -r '.routes[].uri' | rg 'plesk://host/(site/query/docroot|subscription/query/snapshot|dns/query/authority)'
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
- `human_boundary` / granty nietknięte.

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
```bash
# przywróć poprzedni commit w urirun-connector-plesk + core, potem:
docker compose restart urirun-node hr-control hr-bridge
```
HTTP `/plesk/docroot` i stare reality-check bez twinów wracają automatycznie po
cofnięciu kodu; mutacje nie zależą od query twin.

## KPI (ręczny odczyt)
Z reality-check reportów: udział `authority=observed` vs `rule` / `fact_quality=estimated`.
