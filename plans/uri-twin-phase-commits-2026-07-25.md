---
{
  "schema": "subactor.doc/v1",
  "id": "docs.plans.uri-twin-phase-commits-2026-07-25",
  "version": 1,
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

### Pliki (plan)

| URI | Gdzie |
| --- | --- |
| `plesk://host/subscription/query/snapshot` | connector (+ ewentualnie reuse capabilities) |
| `plesk://host/dns/query/authority` | normalizacja istniejącego probe → twin-fact |
| publish / readiness | prefer twin fact nad `last_error` (advisory) |

### Checklista

- [ ] Trzy URI z catalogu uri-twin-plesk widoczne w connector-runtime
- [ ] Reality-check: `last_error` tylko advisory; root cause z twin/DNS/Plesk
- [ ] Żadne usunięcie `human_boundary` / kill switch

---

## Faza 3 — deploy / KPI

- [ ] Runbook: rebuild urirun-node + restart hr-control/bridge
- [ ] Smoke: `subactor` / urirun invoke `site/query/docroot` dla jednej domeny
- [ ] Metryka: % reality-check z `authority=observed` vs rule fallback

---

## Poza zakresem (świadomie)

- Osobne repo `PLESK-DNS` / `PLESK-MODULES`
- Odkręcanie HITL publish
- LLM inventujący URI
