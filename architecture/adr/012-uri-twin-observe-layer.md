---
{
  "schema": "subactor.doc/v1",
  "id": "docs.architecture.adr.012-uri-twin-observe-layer",
  "version": 1,
  "status": "current",
  "updated": "2026-07-25"
}
---

# ADR-012: Warstwa uri-twin (observe) vs urirun connectors (mutate)

- **Status:** Accepted  
- **Data:** 2026-07-25  
- **Kontekst:** false-reality / brak żywego Plesk twin; propozycja org
  `~/github/uri-twin/*`; [`knowledge://subactor/architecture.uri-twin-scope/v1`](../../platform/config/knowledge/entries/architecture.uri-twin-scope.v1.md)

## Decyzja

Wprowadzamy org **uri-twin** jako warstwę **obserwacji** z API URI:

| Warstwa | Odpowiedzialność | Nie robi |
| --- | --- | --- |
| `uri-twin-core` | envelope faktu, freshness, naming | apply, grant, vault secret |
| `uri-twin-plesk` | typed query (subscriptions, docroot, dns) jako packages | mutacje DNS/site |
| `urirun-connector-plesk` | command + dry-run/apply + existing routes | być jedynym SSOT stanu |
| Subactor Control | POLICY, tickets, projekcja HTTP | inventować twin URI |

### URI (kanoniczne)

- Observe: `plesk://host/<resource>/query/<name>`
- Mutate: `plesk://host/<resource>/command/<name>` (pozostaje w connectorze)
- HTTP `/api/connectors/plesk/...` = projekcja, nie SSOT

### Plesk v1 (minimalny zestaw)

1. `plesk://host/subscription/query/snapshot`
2. `plesk://host/site/query/docroot`
3. `plesk://host/dns/query/authority` → normalizacja do `subactor.twin-fact/v1`

### Instancje

Wiele paneli Plesk = wiele **bindings** (`instance_id`, credential handle,
feature flags), jedno repo `uri-twin-plesk`. Nie osobne repo na instalację.

### Repo strategy

Start: **2 repo** (`uri-twin-core`, `uri-twin-plesk`). Nie tworzyć od razu
`PLESK-MODULES`, `PLESK-DNS`, `PLESK-SUBSCRIPTIONS` jako osobnych gitów.

## Konsekwencje

- Publish/readiness muszą czytać twin fact (docroot/topology), nie tylko
  `last_error` ticketu.
- NL2URI mapuje na zatwierdzone query URI w katalogu zdolności.
- Ad‑hoc bridge `GET .../docroot` migruje do URI Process; do czasu migracji
  jest prototypem faktu.
- Pełna autonomia nadal wymaga osobno: ownership ops→bot i granice
  `human_boundary` (ADR-003) — twin tego nie usuwa.

## Powiązane

- ADR-001 (scope / LLM nie tworzy URI)
- ADR-002 (DNS SSOT)
- ADR-004 (publish DoD)
- ADR-011 (digital twin service map)
- `architecture.autonomy-architecture-blockers/v2`
