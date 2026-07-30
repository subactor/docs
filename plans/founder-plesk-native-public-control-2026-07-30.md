---
{
  "schema": "subactor.doc/v1",
  "id": "docs.plans.founder-plesk-native-public-control-2026-07-30",
  "version": 2,
  "status": "current",
  "updated": "2026-07-30"
}
---

# Plan Plesk-native: founder.subactor.com (uri-twin + connector)

## SSOT

Źródłem prawdy jest wyłącznie:

1. **uri-twin** `plesk-surface` (baseline capabilities / workflows / credentials)
2. **Live connector** `urirun-connector-plesk` v0.14 (`plesk://…`)
3. **Manifest** `projekty/founder-subactor-com/project.manifest.json`
4. **Knowledge** `knowledge://subactor/ops.plesk.cloudflare-dns-management/v2`

Nie jest SSOT: zewnętrzny Cloudflare Tunnel, ręczna narracja o routerze Orange
jako domyślna ścieżka deploy.

## Live observation (2026-07-30)

| Probe | URI / payload | Wynik |
| --- | --- | --- |
| Doctor | `plesk://host/doctor/query/report` | `ready`, `production_publish_ready=true`, SFTP+FTP, SSL ensure |
| Methods | `plesk://host/site/query/methods` | sftp+ftp, recommended **sftp** |
| Docroot twin | `plesk://host/site/query/docroot` domain=founder… | **accept**, www_root `/var/www/vhosts/subactor.com/founder.subactor.com` |
| Extensions | `plesk://host/extensions/query/catalog` | **cloudflaredns active** (+ docker, git, ssl…) |
| DNS authority | `plesk://host/dns/query/authority` **zone=`subactor.com`** | provider **cloudflare**, management_plane **cloudflaredns**, consistent |
| DNS records | `plesk://host/dns/query/records` site_id=**185** | A `founder.subactor.com` → `217.160.250.222` |
| DNS propagation | `plesk://host/dns/query/propagation` | **propagated + consensus** (CF/Google/system) |
| Subscription snapshot | `…/subscription/query/snapshot` subscription=`subactor.com` | fresh twin-fact |
| Subscription caps | `…/subscription/query/capabilities` | authenticated; `can_create_domain=false` (**limit_unknown**) |
| Reverse-proxy dry | upstream `https://127.0.0.1:8091` | **fail** `upstream_not_public` (poprawne) |
| Reverse-proxy dry | upstream `https://founder.subactor.com` | **fail** `upstream_loop` (poprawne) |

Service-map DOQL: `tunnel_mode=none`, `public_ingress_mode=plesk_public_origin`,
`dns_sync_extension=cloudflaredns` dla founder.

## Dwa cele (nie mieszać)

### A. Deploy treści / DNS / TLS na originie Plesk — **Plesk-native, działa**

```text
uri-twin observe → plesk:// docroot/methods/dns → dry-run sync (plan_hash)
  → apply grant → plesk://host/site/command/sync (SFTP)
  → EQL: public HTTPS + content hash
```

- DNS: management plane **Plesk + cloudflaredns** → `plesk://host/dns/command/replace`
  (bez tokenu Cloudflare API).
- Static founder: apply już wykonany (6 plików, plan_hash
  `22d7b76de7cb32d8ee790080c639c59d5a22a13f0044bed7ba361eee7349cd87`).
- Public A record i propagation: **zielone**.

### B. Pełny panel Control w Internecie — **nie jest deployem static**

Connector reverse-proxy wymaga **publicznego HTTPS upstreamu z challenge auth
(401/403)**. Lenovo `127.0.0.1` i self-loop na `founder.subactor.com` są
odrzucane by design.

Plesk-native warianty (z twin, bez Zero Trust jako domyślności):

1. **Zachować `static_loopback_bridge`** (obecny kontrakt): publiczny Plesk =
   landing + action; panel na maszynie Foundera po tokenie w fragmencie.
   Autonomia deployu = tor A. **To jest zgodne z manifestem i twin.**
2. **Uruchomić aplikację Control na hostingu osiągalnym z Internetu**
   (np. kontener/Docker na serwerze Plesk — extension `docker` jest w katalogu
   — albo osobna subskrypcja/domena), z Basic Auth; potem
   `reverse-proxy-ensure` z dry-run → grant → apply (`PLESK_REVERSE_PROXY_APPLY`).
3. Managed outbound tunnel — **opcjonalny** (manifest `tunnel_mode=none`);
   nie zastępuje toru A.

## Plan wykonania (kolejność)

### P0 — Domknąć tor A (już w większości)

- [x] Live doctor / methods / docroot / extensions
- [x] DNS authority + records + propagation (poprawne payloady: `zone`, `site_id`)
- [x] Static sync apply + public content match
- [x] Payload SSOT (`plesk-uri-payloads.mjs`): zone / site_id / subscription / subdomain
- [ ] DNS dry-run replace (expect `changed=false`) jako stały smoke w control
- [ ] Reality-check: nie oznaczać `converged` jako „Control panel ready”

### P1 — Publiczny panel (tylko jeśli Founder wymaga B)

1. Twin: wybrać capability `plesk.application.runtime` / reverse-proxy z
   `required_capabilities` (root SSH CLI).
2. Wystawić **publiczny** origin HTTPS z 401 (nie loopback).
3. `plesk://host/site/command/reverse-proxy-ensure` dry-run → plan_hash.
4. Apply dopiero z grantem + `PLESK_REVERSE_PROXY_APPLY=1`.
5. EQL: unauth 401, auth 200, action 200, TLS SAN.

### P1 — Autonomia pętli

- Consumers: tylko one-shot lease, nie permanent=1 z `.env` bez overlay safe.
- [x] Naprawić shape payloadów w producentach ticketów (dns zone, subscription name) — `plesk-uri-payloads.mjs`.
- Capability inventory API (dziś 404) — projekcja doctor+extensions+methods.
- `subscription_domain_limit_unknown` — nie planować nowych domen bez limitu.

## Payloady (żeby twin queries nie padały 422)

```json
// DNS authority — zone, nie hostname
{"zone": "subactor.com"}

// DNS records — site_id z manifestu
{"site_id": 185, "host": "founder.subactor.com"}

// Subscription
{"subscription": "subactor.com"}
```

## Werdykt

| Pytanie | Odpowiedź z twin/live |
| --- | --- |
| Czy deploy founder na Plesku jest autonomiczny w torze A? | **Tak** (observe + granted apply) |
| Czy potrzeba tokenu Cloudflare Tunnel do DNS/deploy? | **Nie** (cloudflaredns w Plesku) |
| Czy publiczny pełny Control jest zrobiony? | **Nie** — brak publicznego HTTPS upstreamu; fail-closed reverse-proxy jest poprawny |
| Co psuje „pełną autonomię” kolejki? | gates, waiting_input, false-reality converged, luki payloadów |

## Następny mały krok implementacyjny

1. Smoke test DNS replace dry-run dla site 185 (expect no change).
2. Poprawić producers ticketów / remediation, by używały `zone` + `site_id`.
3. Dopiero potem decyzja Foundera: zostać przy static_loopback_bridge **albo**
   fundować publiczny origin Control na Plesku (Docker/subscription app).
