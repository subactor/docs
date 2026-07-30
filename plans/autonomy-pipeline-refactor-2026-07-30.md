---
{
  "schema": "subactor.doc/v1",
  "id": "docs.plans.autonomy-pipeline-refactor-2026-07-30",
  "version": 1,
  "status": "current",
  "updated": "2026-07-30"
}
---

# Plan refaktoru toru autonomii (NL→…→EQL) — 2026-07-30

## Cel

Doprowadzić producentów ticketów i diagnostyki do **jednego SSOT payloadów**
Plesk zgodnego ze snapshotem bindings, udokumentować kanoniczny tor wykonania
oraz domknąć testami to, co wcześniej dawało `bridge_422` (złe nazwy pól).

## Luki vs stan kodu (przed tym slice)

| Luk | Objaw | Gdzie |
| --- | --- | --- |
| `subdomain-ensure` z `domain`/`subscription`/`origin_ip` | 422 / research `payload_invalid` | `domain-readiness-diagnosis`, `project-remediation` |
| DNS authority z `hostname` zamiast `zone` | 422 na live | historyczne ręczne wywołania; remediate już często OK |
| `subscription/query/*` z `domain` | nieznany param w snapshot | step input w strategy catalog + plan merge |
| Brak centralnego buildera | dryf aliasów między producentami | brak `plesk-uri-payloads.mjs` |
| Twin helper tylko docroot | DNS/subscription normalize ad-hoc | `plesk-twin-sources.mjs` |
| Research branch nie wpięty w remediation create | eskalacja bez schema check | `ensureProjectRemediationTickets` (P1) |
| Kill switch consumers/mutations | 0 automatic execute | celowe fail-closed, nie bug |

## Slice P0 (ten commit / ta sesja)

1. **Dokument kanoniczny** `docs/architecture/autonomy-execution-pipeline.md`.
2. **Knowledge** `architecture.autonomy-execution-pipeline.v1` (wewnętrzny SSOT narracji).
3. **Helper** `core/services/control/src/plesk-uri-payloads.mjs`:
   - authority / records / replace / reconcile / propagation
   - subscription query (`subscription`, nie `name`/`domain` w wire)
   - subdomain-ensure (`subdomain` + `parent_domain` + `www_root`)
   - domain-ensure, observe-then-dry-run steps
4. **Producenci** używają helpera: diagnosis + DNS remediation remap.
5. **Twin**: `twinFromDnsAuthority`, `twinFromSubscriptionSnapshot`, coverage hooks.
6. **Testy jednostkowe** payload + twin + diagnosis + remediation founder path.

### Kryteria akceptacji P0

- [x] `subdomain-ensure` payload bez `domain`/`subscription`/`origin_ip`
- [x] replace: `site_id` + `host` + record; authority: `zone`
- [x] subscription observe: tylko `subscription`
- [x] testy zielone w `core/services/control` dla nowych i dotkniętych plików
- [ ] (opcjonalnie live) dry-run DNS replace `changed=false` read-only — bez apply

## Slice P1 (następne)

1. Wpiąć `researchPayload` / `researchBlocker` **przed** create remediation ticket
   (fail-closed przy unknown params; nie eskaluj 422, które da się przewidzieć).
2. Strategy catalog: usunąć `domain` z inputów subscription steps u źródła
   (dziś sanitizacja w `project-remediation` po `buildCapabilityPlan`).
3. Snapshot refresh job: automatyczny zrzut bindings z zainstalowanego connectora.
4. Coverage KPI: tickety DNS bez `dns/query/authority` w planie = `legacy_path`.
5. Smoke script read-only: doctor + authority(zone) + records(site_id) +
   subscription(snapshot) + DNS replace dry-run.

## Slice P2

1. Full Control public origin (nie static landing) — tylko po public upstream + auth.
2. Research → auto replan gdy verdict `renamed_or_mistyped` / `payload_invalid`.
3. Jednolity `input_hash` w receipt obok `plan_hash` dla wszystkich command routes.

## Zasady refaktoru

- Małe commity, bez mieszania kill-switch policy z bugfixami payloadów.
- Nie dodawać Cloudflare Tunnel jako domyślnej ścieżki founder/public site.
- Nie obniżać bramek `apply` / grant / AQL w imię „autonomii”.
- Po zmianie managed docs: `npm run artifacts:build` + `artifacts:check` w Platform.

## Powiązane

- [autonomy-execution-pipeline.md](../architecture/autonomy-execution-pipeline.md)
- [founder-plesk-native-public-control-2026-07-30.md](./founder-plesk-native-public-control-2026-07-30.md)
- [autonomy-loop-and-twin.md](../architecture/autonomy-loop-and-twin.md)
