---
{
  "schema": "subactor.doc/v1",
  "id": "docs.operations.digital-twin-service-map-operations-guide",
  "version": 2,
  "status": "current",
  "updated": "2026-07-24"
}
---

# Przewodnik operacyjny: Digital Twin Service Map (public site)

## Cel

Powtarzalna diagnostyka i utrzymanie **bindingu public site** (Plesk + cloudflaredns, `tunnel_mode=none`) oraz kontrolera ticketów, który:

- **nie** tworzy managed-tunnel credential przy formalnym zakazie tunelu,
- **anuluje** (supersede) aktywne tickety credential,
- **odblokowuje** tickety źródłowe na trasę `application_route_not_ready`,
- **nie myli** default page Pleska z brakiem DNS ani z „potrzebą tunelu”.

Kanoniczna wiedza: `knowledge://subactor/architecture.digital-twin.public-site-service-map/v4`.

---

## 1. Warstwy (kolejność czytania)

| # | Warstwa | Artefakt |
|---|---------|----------|
| 1 | Intent | `platform/config/intent-contracts/founder-public-site-binding.v1.json` |
| 2 | DOQL mapa | `platform/config/digital-twin/public-site-service-map.doql.json` |
| 3 | DOQL inventory | `platform/config/digital-twin/public-site-capability-inventory.doql.json` |
| 4 | Strategy | `project.application.route.plesk-origin`, `project.public-site.bootstrap.plesk-origin` w `problem-strategies/catalog.v1.json` |
| 5 | Manifest projektu | `projekty/*/project.manifest.json` (`public_ingress_mode`, `tunnel_mode`, `dns_*`) |
| 6 | Runtime control | `core/services/control/src/public-site-*.mjs`, etap `managed_tunnel_followup` |

---

## 2. Weryfikacja mapy usług (read-only)

```bash
cd platform
node scripts/run-public-site-service-map.mjs
```

Control API (scope `projects:read`):

```http
GET /api/projects/public-site-service-map
GET /api/projects/public-site-service-map?live=1
```

### Oczekiwane assessments (portfolio z plesk origin)

- `dns_management_binding`: `plesk_cloudflaredns_complete` (gdy wszystkie projekty deklarują Plesk+cloudflaredns)
- `tunnel_policy`: `consistent` (brak konfliktu `tunnel_mode=none` vs `managed_outbound_tunnel`)
- `founder_public_site_binding`: `selected_plesk_public_origin`

`founder_bootstrap_playbook`: lista ~11 kroków inventory → domain/DNS/TLS → dry-run treści → verify.

`live=1` próbuje doctor Plesk przez URI; przy błędzie fixture (fail-closed, bez mutacji).

---

## 3. Cykl kontrolera (supersede / unblock)

```http
POST /api/tickets/lifecycle/run
```

(scope `routing:manage`)

### Etap `managed_tunnel_followup`

1. Załaduj projekty z `SUBACTOR_PROJECTS_PATH` (w kontenerze: host path zamontowany + `/workspace/projekty`).
2. `managedTunnelSupersedeCandidates` — **tylko** tickety credential intake.
3. Cancel + `supersession_reason=public_site_binding_tunnel_mode_none`.
4. Source unblock: `blocked-by` / `blocked_by:CHILD` → `application_route_not_ready`, label `binding:tunnel_mode_none`.
5. Skip nowych credential intake dla źródeł covered by `tunnel_mode=none`.

### Audit

```text
public_site.tunnel_binding_superseded
public_site.tunnel_binding_supersede_failed
public_site.projects_load_failed
public_site.projects_load_empty
```

### Live reference (2026-07-24)

| Ticket | Oczekiwany stan |
|--------|-----------------|
| PLF-1272 | `canceled`, supersede tunnel binding |
| PLF-592 | `open`, `application_route_not_ready`, `managed_tunnel_superseded=PLF-1272` |
| PLF-894 / PLF-893 | `open`, `application_route_not_ready` (treść/trasa, **nie** tunnel) |

---

## 4. Remediacja `application_route_not_ready` przy plesk origin

Przy `tunnel_mode=none` / `public_ingress_mode=plesk_public_origin`:

- **nie** dodawaj zależności managed-tunnel,
- **nie** wymagaj kroku `provide-upstream` (free-form),
- użyj strategy `project.application.route.plesk-origin` (inspect → confirm-binding → dry-run → approve → verify),
- stare envelope z `provide-upstream` są **stale** (`remediationContractStaleForBinding`) i powinny być przepisane w cyklu remediacji.

Root cause publicznego default page: **brak kontraktu treści/upstreamu**, nie brak tunelu.

---

## 5. Deploy control (unikanie stale image)

W `platform/docker-compose.yml` serwis `hr-control` montuje:

```yaml
- ../core/services/control/src:/app/services/control/src:ro
```

Po zmianach w `core/services/control/src` wystarczy **restart** kontenera (bez rebuild), o ile mount jest aktywny:

```bash
cd platform && docker compose up -d hr-control
```

Produkcyjny bake obrazu nadal zawiera te same pliki; mount jest warstwą ops/dev.

---

## 6. Audyt ticketów Planfile

```bash
cd platform
node scripts/ticket-auditor.mjs --limit 20
```

Szukaj aktywnych:

- `managed-tunnel-credential*` przy projektach z `tunnel_mode=none` → błąd policy,
- `blocked-by:PLF-*` na anulowanym credential child → brak unblock,
- `provide-upstream` w remediation founder przy plesk origin → stale contract.

---

## 7. Artefakty i testy

```bash
cd platform
npm run artifacts:build
npm run artifacts:check

cd ../core/services/control
node --test tests/public-site-binding-followup.test.mjs \
  tests/public-site-service-map.test.mjs \
  tests/project-reconciliation.test.mjs
```

---

## 8. Znane ograniczenia

1. **Pełna** `POST /api/projects/reconciliation/run` może przekraczać 300s na całym portfelu — nie mylić timeoutu z błędem bindingu.
2. Planfile bywa wolny/unhealthy pod obciążeniem — retry lifecycle, nie force-push mutacji.
3. Digital Twin **nie** naprawia automatycznie reverse-proxy/upstream; daje właściwą trasę diagnostyczną i dry-run.
4. Apply produkcyjny nadal wymaga `plan_hash` + master gate.

---

## 9. Szybki checklist „czy jest OK?”

- [ ] Service-map: `founder_public_site_binding=selected_plesk_public_origin`
- [ ] Brak aktywnych ticketów `managed-tunnel-credential` dla founder
- [ ] PLF-1272 (lub następca credential) terminal canceled z `public_site_binding_tunnel_mode_none`
- [ ] PLF-592 / remediation: `application_route_not_ready`, bez `blocked-by` na tunnel child
- [ ] Remediation process: **brak** `provide-upstream` przy `tunnel_mode=none`
- [ ] HTTPS founder: DNS/TLS OK; treść to osobny blocker (nie tunnel)
