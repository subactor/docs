---
{
  "schema": "subactor.doc/v1",
  "id": "docs.architecture.adr.012-uri-twin-observe-layer",
  "version": 3,
  "status": "current",
  "updated": "2026-07-29"
}
---

# ADR-012: Warstwa uri-twin (observe) vs urirun connectors (mutate)

- **Status:** Accepted  
- **Data:** 2026-07-25  
- **Kontekst:** false-reality / brak żywego Plesk twin; propozycja org
  `~/github/uri-twin/*`; [`knowledge://subactor/architecture.uri-twin-scope/v1`](../../../platform/config/knowledge/entries/architecture.uri-twin-scope.v1.md)

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

Start: **2 publiczne repo**
([`uri-twin-core`](https://github.com/uri-twin/uri-twin-core),
[`uri-twin-plesk`](https://github.com/uri-twin/uri-twin-plesk)). Nie tworzyć od
razu `PLESK-MODULES`, `PLESK-DNS`, `PLESK-SUBSCRIPTIONS` jako osobnych gitów.

### Git baseline + API overlay

- Konektor ładuje baseline bezpośrednio z Git przy starcie i zapisuje dokładny
  commit oraz digest treści.
- Brak sieci nie rozszerza uprawnień: używany jest zwalidowany last-known-good,
  a następnie osadzony fallback oznaczony jako `stale`.
- Obserwacja API może dodać instancje zatwierdzonych typów zasobów, potwierdzić
  flagę lub zablokować capability. Nie może dodać wykonywalnej trasy.
- Nieznane moduły są widoczne jako `review-required` i wymagają zmiany baseline'u
  w Git.

### Graf wielu konektorów

Workflow wskazuje wymagane capability, a capability może mieć kilku providerów.
Resolver wybiera provider na podstawie aktywnego manifestu tras, bindingu
środowiska i nazw uchwytów poświadczeń. Przykład DNS wybiera Plesk/
`cloudflaredns` albo Namecheap bez przekazywania wartości sekretu do twina.

Decyzja `refuse` lub nieświeży twin-fact blokuje zależną komendę jako
`precondition-blocked`. Dzięki temu spodziewany problem nie staje się najpierw
błędem konektora i nowym ticketem.

## Konsekwencje

- Publish/readiness muszą czytać twin fact (docroot/topology), nie tylko
  `last_error` ticketu.
- NL2URI mapuje na zatwierdzone query URI w katalogu zdolności.
- Ad‑hoc bridge `GET .../docroot` migruje do URI Process; do czasu migracji
  jest prototypem faktu.
- Pełna autonomia nadal wymaga osobno: ownership ops→bot i granice
  `human_boundary` (ADR-003) — twin tego nie usuwa.
- `map_hash` ma identyczną, kanoniczną postać w implementacji JS i Python;
  zmiana opisu nie zmienia tożsamości mapy, a zmiana decyzji wykonawczej tak.

## Weryfikacja 2026-07-29

- `uri-twin-core` `v0.2.3`, `uri-twin-plesk` `v0.2.2`, baseline Plesk `v3`.
- Start konektora pobrał baseline z Git z commita
  `069008bff9c08366d2e3e156c4c9b5b937eaa37e` i zweryfikował digest.
- Żywy Plesk zwrócił świeże fakty dla 19 subskrypcji, docroot i DNS oraz katalog
  rozszerzeń; wszystkie cztery query były obecne w rejestrze 644 tras.
- `observe-topology` rozwiązał plan 2/2. Obserwowany docroot z decyzją `refuse`
  zatrzymał `publish-site-with-managed-dns` przed wykonaniem z gapem
  `fact_refused: plesk.site.docroot`; nie uruchomiono żadnej komendy.

## Powiązane

- ADR-001 (scope / LLM nie tworzy URI)
- ADR-002 (DNS SSOT)
- ADR-004 (publish DoD)
- ADR-011 (digital twin service map)
- `architecture.autonomy-architecture-blockers/v2`
