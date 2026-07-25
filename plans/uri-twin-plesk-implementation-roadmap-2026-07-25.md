---
{
  "schema": "subactor.doc/v1",
  "id": "docs.plans.uri-twin-plesk-implementation-roadmap-2026-07-25",
  "version": 3,
  "status": "current",
  "updated": "2026-07-25"
}
---

# Plan implementacji: Plesk URI Twin (observe, bez zmian polityki hitl)

## Cel
Wersjonowany, kanałowo stabilny, SSOT dla żywego stanu Plesk jako warstwa **observe**, przy zachowaniu istniejącego modelu urirun mutacji i Control governance.

## Założenia projektowe (zgodne z Twoją notatką)
1. Nie tworzymy osobnych repo na typy Plesk (`PLESK`, `PLESK-MODULES`, `PLESK-DNS`, `PLESK-SUBSCRIPTIONS`) w fazie 1.
2. Tworzymy obserwacyjny twin (query/snapshot) niezależny od apply/grant.
3. Wydawanie faktów jest kanałem `query` URI (`plesk://host/.../query/...`), a HTTP w Control pozostaje tylko projekcją.
4. Jedna logika twin-ów działa dla wielu instancji przez `instance_id` + binding/credential handle + feature flags.

## Fazy realizacji

### Faza 0 — stabilizacja podstawy (już częściowo wykonana)
- [x] Opis architektoniczny w ADR: [012-uri-twin-observe-layer.md](../architecture/adr/012-uri-twin-observe-layer.md)
- [x] Knowledge: `knowledge://subactor/architecture.uri-twin-scope/v1`
- [x] Szkielet org `~/github/uri-twin/` (`uri-twin-core`, `uri-twin-plesk` + catalog v1)
- [x] Weryfikacja problemu: ad-hoc `GET /api/connectors/plesk/docroot` i brak jednolitego SSOT
- [x] Przygotowanie prototypu query-faktu docroot w bridge: `GET /plesk/docroot`
- [x] Projekcja HTTP do Control: `GET /api/connectors/plesk/docroot`

### Faza 1 — migracja do prawdziwego URI twin (v1)
- [x] Handler `plesk://host/site/query/docroot` w `urirun-connector-plesk` (+ manifest/README/test)
- [x] Envelope `subactor.twin-fact/v1` + `snapshot_hash` / `fact_quality`
- [x] Reality-check czyta docroot twin (bridge projekcja → `sources.plesk_docroot_twin`); `last_error` nie jest SSOT topologii
- [x] `plesk://host/subscription/query/snapshot` jako twin-fact
- [x] `plesk://host/dns/query/authority` normalizacja do twin-fact
- [x] Plan faz/commitów: [uri-twin-phase-commits-2026-07-25.md](./uri-twin-phase-commits-2026-07-25.md)
- [x] fail-open: docroot/subscription → `estimated`; dns inconsistent → `stale` + fail kodem authority

### Faza 2 — spójność planowania i readiness
- [x] Reality-check findings dla subscription/dns twins (advisory; bez usuwania HITL)
- [ ] publish/readiness recipes wymuszają dokumentację docroot fact w planie
- [ ] Testy E2E check-run na żywym panelu po deployu connectora

### Faza 3 — przygotowanie do autonomii
- [ ] Nie usuwamy blokad `human_boundary`; mapujemy je do konkretnych klas ryzyka:
  - publish dry-run: auto
  - apply + ryzyko R2: requires grant
  - only-operator/owner steps: founder/tenant human path
- [ ] Dodać KPI testów: % publish opartych o aktualny fact vs legacy path
- [ ] Utworzyć runbook deploya twin i roll-back.

## Minimalny podział odpowiedzialności
- `uri-twin-core`:
  - envelope faktu (`subactor.twin-fact/v1`), freshness, schema/validation, metryki.
- `uri-twin-plesk`:
  - query dla subscriptions/docroot/dns, binding do wielu instancji.
- `urirun-connector-plesk`:
  - pozostaje jedynym kanałem mutacji (`command`, dry-run/apply/grant).
- `Subactor Control`:
  - POLICY/HITL/reality orchestration; HTTP jako projekcja istniejących URI.

## Ryzyka operacyjne
- Największe ryzyko to niespójność danych między live state (twin) a cache-control; to zamykać przez `snapshot_hash` + TTL.
- Bez explicit bindingu `instance_id` i feature flags twiny będą miały zbyt słabą izolację między panelami.
- Brak restartu usługi po wdrożeniu kodu => endpointy i registry mogą pozostać w stanie sprzecznym; każdy deploy musi uwzględniać rebuild/restart.

## Odbiór i akceptacja
- 1) `plan` nie wykonuje żadnej mutacji opartych o obserwację;
- 2) `site/query/docroot` działa przez URI i jest czytelny w `/api/connector-runtime` routes;
- 3) `publish`/`reality-check` pokazują fakty twin zamiast jedynie błędów ticketów;
- 4) `human_boundary` pozostaje aktywny tam, gdzie model ryzyka wymaga człowieka.
