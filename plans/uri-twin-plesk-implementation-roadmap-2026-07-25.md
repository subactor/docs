---
{
  "schema": "subactor.doc/v1",
  "id": "docs.plans.uri-twin-plesk-implementation-roadmap-2026-07-25",
  "version": 9,
  "status": "current",
  "updated": "2026-07-29"
}
---

# Plan implementacji: Plesk URI Twin (observe, bez zmian polityki hitl)

**Stan realizacji 2026-07-29:** fazy 0–3 są wykonane. Plan pozostaje częściowy,
ponieważ jeden warunkowy element federacji z fazy 4 jest nadal otwarty. Katalog,
attestacje i generator propozycji review-required są wykonane.

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
- [x] publish/readiness recipes wymuszają dokumentację docroot fact w planie
- [x] Testy E2E check-run na żywym panelu po deployu connectora

### Faza 3 — przygotowanie do autonomii
- [x] Nie usuwamy blokad `human_boundary`; mapujemy je do konkretnych klas ryzyka:
  - publish dry-run: auto
  - apply + ryzyko R2: requires grant
  - only-operator/owner steps: founder/tenant human path
- [x] Dodać KPI testów: % publish opartych o aktualny fact vs legacy path —
  `reality-check.metrics.publish_twin_coverage` rozdziela `current_fact`,
  `legacy_path` i `stale_or_unverified`; aktualny fakt wymaga jakości `fresh`,
  znacznika czasu i wieku nieprzekraczającego 15 minut.
- [x] Utworzyć runbook deploya twin i roll-back: [uri-twin-plesk-deploy-runbook.md](../operations/uri-twin-plesk-deploy-runbook.md)
- [x] Opublikować publiczne repo `uri-twin-core` i `uri-twin-plesk` z CI i
  wersjonowanymi tagami.
- [x] Ładować Plesk baseline z Git jako primary; zwracać commit/digest i używać
  zwalidowanego cache/embedded wyłącznie jako oznaczony fallback.
- [x] Dodać mapę środowiska, usług, typów zasobów i nazwanych workflow oraz
  bezpieczne wzbogacanie przez dane API.
- [x] Dodać alternatywnych providerów i zależności wielu konektorów (Plesk/
  `cloudflaredns` albo Namecheap dla DNS).
- [x] Blokować command przed dispatch, gdy twin-fact ma decyzję `refuse` albo
  nieświeżą jakość (`precondition-blocked`).

### Faza 4 — federacja twinów
- [x] Katalog organizacji `uri-twin` z maszynowo czytelnym indeksem repo →
  family → baseline → obsługiwane schematy: publiczne `uri-twin-catalog` v0.1.0
  ma CI i conformance względem tagów/pakietów/baseline. Loader twin-map
  rozwiązuje domyślne źródło przez katalog, ogranicza repo do organizacji,
  zapisuje provenance katalogu i zachowuje direct/cache/embedded fallback.
- [x] Podpisane wydania/attestacje baseline i polityka dozwolonych signerów w
  loaderze: Plesk v0.2.4 ma odłączoną attestację Ed25519, CI/offline verify,
  signer secret w Actions i publiczny fingerprint przypięty w consumerze.
- [x] Generator propozycji PR z `review-required` discoveries: deterministyczny
  draft ma `authority_change: none`, filtr sekretów i wymagane review,
  manifest-route conformance oraz nową attestację.
- [ ] Dodać kolejne platformy jako osobne repo twin dopiero przy realnym
  konektorze i teście manifest-route conformance.

## Minimalny podział odpowiedzialności
- `uri-twin-core`:
  - envelope faktu (`subactor.twin-fact/v1`), freshness, schema/validation, metryki.
- `uri-twin-plesk`:
  - środowisko, services/resources/workflows, query dla
    subscriptions/docroot/dns i binding do wielu instancji.
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
- 5) Git provenance jest jawne, JS/Python obliczają ten sam `map_hash`, a URI
  providerów istnieją w aktywnych manifestach konektorów.
