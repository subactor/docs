---
{
  "schema": "subactor.doc/v1",
  "id": "docs.ops.raport-autonomia-plesk-subactor-2026-07-25",
  "version": 1,
  "status": "current",
  "updated": "2026-07-25"
}
---

# Raport: Autonomia Subactor + Plesk connector (status 2026-07-25)

## Status wdrożenia (krok „naprawa plesk twin”)

- Bridge: `GET /plesk/docroot` → fakt `plesk.docroot.decision/v1` (`docrootDecisionFact`), best-effort XML API.
- Control: projekcja `GET /api/connectors/plesk/docroot?...`.
- Do czasu restartu runtime produkcja może zwracać 404 na nowej trasie HTTP.

## Cel sprawdzenia

- Czy publikacja jest w pełni autonomiczna?
- Czy Plesk connector kończy publish?
- Czy potrzebny jest Plesk Twin (żywy stan instancji)?

## Obserwacje live

| Probe | Wynik |
| --- | --- |
| `GET /api/connectors` | `plesk` configured/enabled |
| `GET /api/connector-runtime` | trasy sync/publish/publish-verify/remote-inventory obecne |
| `site/query/docroot` w runtime | brak aktywnej trasy URI (lukę łata prototyp HTTP) |
| reality-check `autonomicznosc.pl` / `docs.subactor.com` | `ready_to_publish`, blocker `human_boundary_requires_child_ticket:publish` (PLF-1052, PLF-888) |
| plans `proposed` | ~155, część z `human_approval` |

## Wnioski

1. **Autonomia nie jest pełna** — publish strukturalnie gotowy, governance (`human_boundary`) wymusza człowieka.
2. **Plesk connector wymaga warstwy observe twin** — docroot/topology jako `plesk://…/query/…` + `subactor.twin-fact/v1`, nie tylko ad-hoc HTTP.
3. HTTP `/docroot` jest **prototypem faktu**; SSOT docelowo ADR-012 / uri-twin.

## Powiązane

- ADR-012 uri-twin observe layer
- `knowledge://subactor/architecture.uri-twin-scope/v1`
- Plan: [`../plans/uri-twin-plesk-implementation-roadmap-2026-07-25.md`](../plans/uri-twin-plesk-implementation-roadmap-2026-07-25.md)
- Org: `~/github/uri-twin/`
