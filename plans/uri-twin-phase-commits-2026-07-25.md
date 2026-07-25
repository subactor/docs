---
{
  "schema": "subactor.doc/v1",
  "id": "docs.plans.uri-twin-phase-commits-2026-07-25",
  "version": 5,
  "status": "current",
  "updated": "2026-07-25"
}
---

# Plan faz / commitów: uri-twin Plesk (bez PR)

**Cel:** SSOT observe przez `plesk://…/query/…` + twin-fact; Control czyta fakty;
mutacje i `human_boundary` bez zmian.  
**ADR:** [012](../architecture/adr/012-uri-twin-observe-layer.md)  
**Roadmapa:** [uri-twin-plesk-implementation-roadmap](./uri-twin-plesk-implementation-roadmap-2026-07-25.md)

Ship: commit na `main` w danym repo (nie otwieramy PR).

---

## Faza 0 — done

| Commit / artefakt | Repo |
| --- | --- |
| ADR-012 + knowledge uri-twin-scope | `docs`, `platform` |
| Szkielet `@uri-twin/core` + `@uri-twin/plesk` | `~/github/uri-twin/*` |
| Prototyp HTTP `/plesk/docroot` | `connectors` bridge |

**Akceptacja:** dokumenty + `npm test` w uri-twin; HTTP projekcja nie jest SSOT.

---

## Faza 1 — URI SSOT `site/query/docroot` (ten cykl)

### Pliki

| Plik | Zmiana |
| --- | --- |
| `urirun-connectors/urirun-connector-plesk/urirun_connector_plesk/core.py` | handler `site/query/docroot` → `subactor.twin-fact/v1` |
| `…/connector.manifest.json` | dodać route |
| `…/README.md` | wiersz URI |
| `…/tests/test_plesk.py` | rejestr trasy + unit z fake XML |
| `core/…/project-reality-check.mjs` | findings z twin docroot (advisory vs last_error) |
| `core/…/routes/project-reconciliation.mjs` | pobierz `/plesk/docroot` do reportu |
| `core/…/tests/project-reality-check.test.mjs` | finding docroot |
| `docs/plans/uri-twin-plesk-implementation-roadmap-…` | odhaczyć Faza 1 partial |

### Checklista akceptacji

- [ ] `plesk://host/site/query/docroot` w manifeście i w liście handlerów testu
- [ ] Wynik ma `schema: subactor.twin-fact/v1`, `twin_type: plesk.site.docroot`, `snapshot_hash`, `uri` z `/query/`
- [ ] Brak danych → `ok` z `fact_quality: estimated|stale` **albo** fail kodem observe — **bez** mutacji, **bez** grantu
- [ ] Reality-check report zawiera `sources.plesk_docroot_twin` i nie zastępuje `human_boundary`
- [ ] pytest zielony; control unit test zielony
- [ ] Commit na `main` w `urirun-connector-plesk` + `core` (+ docs)

---

## Faza 2 — pozostałe query v1 + Control SSOT

### Pliki

| URI / obszar | Stan |
| --- | --- |
| `plesk://host/subscription/query/snapshot` | **done** (connector + twin-fact) |
| `plesk://host/dns/query/authority` | **done** (twin-fact na istniejącym probe) |
| reality-check findings | **done** (advisory subscription/dns twins) |
| publish recipe wymusza docroot fact | **done** (site-publish + docs/www/logo step-catalog) |
| reality-check via URI Process | **done** (`?live=1` / `?via=uri`, bridge fallback) |
| live E2E po deployu | **done** (restart + smoke 2026-07-25) |

### Checklista

- [x] Trzy URI z catalogu uri-twin-plesk w connectorze
- [x] Reality-check: twin findings; `human_boundary` nietknięty
- [x] Publish/readiness czyta URI Process (nie tylko bridge HTTP)
- [x] Żadne usunięcie kill switch / HITL

---

## Faza 3 — deploy / KPI

- [x] Runbook: [uri-twin-plesk-deploy-runbook.md](../operations/uri-twin-plesk-deploy-runbook.md)
- [x] Smoke: `subactor uri plesk://host/site/query/docroot` + `reality-check?via=uri` (docs.subactor.com)
- [x] Metryka: % reality-check z `authority=observed` vs rule fallback — live 2026-07-25: **60% observed (3/5)** na *.subactor.com; szczegóły w runbooku

---

## Poza zakresem (świadomie)

- Osobne repo `PLESK-DNS` / `PLESK-MODULES`
- Odkręcanie HITL publish
- LLM inventujący URI
