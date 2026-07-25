---
{
  "schema": "subactor.doc/v1",
  "id": "docs.operations.false-reality-and-shell-verification-2026-07-24",
  "version": 2,
  "status": "current",
  "updated": "2026-07-24"
}
---

# Fałszywe obrazy rzeczywistości i weryfikacja shell/REST/URI

## Problem (przykład PLF-884)

Founder Attention pokazywał **PLF-884** („Wystaw i zweryfikuj TLS — contracts.subactor.com”) jako **DZIAŁAJ TERAZ**, podczas gdy:

| Warstwa obserwacji | Fakt |
|---|---|
| Public DNS (1.1.1.1) | brak A dla `contracts.subactor.com` |
| Plesk REST `/api/v2/domains?name=contracts.subactor.com` | `[]` — **brak obiektu** w panelu |
| Reconciliation | `dns_not_ready` → `tls_not_ready` → `application_route_not_ready` |
| Tickety | **PLF-883** = DNS (root), **PLF-884** = TLS `blocked-by:PLF-883` |

Czyli system prosił o decyzję SSL, zanim istniała subdomena w Plesk i rekord DNS. To klasyczny **fałszywy obraz rzeczywistości**: ticket/attention ≠ live observation.

---

## Skąd biorą się fałszywe obrazy

### 1. Ticket state ≠ causal chain (Digital Twin diagnosis)

Reconciliation buduje `diagnosis.causal_chain`:

```text
public_dns_name_unresolved
  → acme_domain_validation_unavailable
  → tls_certificate_issuance_blocked
  → strict_https_unavailable
```

Ale remediacja tworzy **równoległe** tickety DNS + TLS + app, a attention bierze **każdy** `queue=founder` + `waiting_input`. TLS dostaje `readiness_preflight:…ensure-ssl` mimo `blocked-by:PLF-883`.

**Lek:** attention ignoruje `blocked-by` / `blocked_by:PLF-*` (`founder-attention-policy`). Root ticket (DNS) jest jedynym founder/bot action.

### 2. Snapshot recon vs live dig

`project-reconciliation.json` ma `observed_at` i może być stary. UI/ops-observer pokazują ticket, nie świeży dig.

**Lek:** `subactor reality <domain>` łączy dig + recon + tickety + attention w jednym raporcie.

### 3. Lab REST na live Plesk

Operacje `example.test` szły na `PLESK_MODE=live` → HTML 404 na `/api/v2/mail/mailboxes` i `/sites/publish` (mock-only).

**Lek:** lab→mock routing (`plesk-lab-route.mjs`); live mailbox→CLI; live publish→domain-fs.

### 4. Process envelope / preserveContract

Stary process z `provide-upstream` lub ensure-ssl przeżywał mimo `tunnel_mode=none` / DNS missing.

**Lek:** `remediationContractStaleForBinding`; DNS plesk path → `project-operator-bot` + subdomain-ensure.

### 5. Planfile latency / partial reads

Timeouty Planfile dają UI z niekompletnym stanem → wrażenie „system nie wie”.

**Lek:** reality-check z timeoutami i jawnym `source.ok=false`; nie traktować partial 404/timeout jako „domena OK”.

### 6. Mieszanie warstw policy i execution

Service-map mówi `tunnel_mode=none`, a ticket nadal ma tunnel blocker. Policy (Intent/DOQL) ≠ execution ticket.

**Lek:** supersede tunnel tickets; reality-check flags mismatch policy vs ticket labels.

---

## Rozwiązania na bazie Digital Twin / DSL

| Warstwa | Rola w prawdzie |
|---|---|
| **Intent** | Co wolno (np. bez tunnel) |
| **DOQL service-map** | Portfolio binding, konflikty |
| **domain-readiness diagnosis** | Causal chain DNS→TLS→HTTPS + `next_processes` |
| **Strategy DSL** | Kroki (ensure-domain / subdomain-ensure / ssl) |
| **project.manifest** | `origin_ip`, subscription, gates |
| **Ticket + SODL** | Plan wykonania — musi cytować observation |
| **Attention policy** | Kto dostaje „DZIAŁAJ TERAZ” |

Zasada: **observation receipt bije ticket last_error**. Jeśli dig/Plesk mówią „brak domeny”, ticket TLS nie może prosić o SSL.

`next_processes` przy `public_dns_name_unresolved` dla multi-label hosta na Plesk:

1. `plesk://host/site/command/subdomain-ensure` (`apply=false`)
2. `plesk://host/dns/query/propagation`
3. `plesk://host/dns/command/replace` (`apply=false`)

---

## Testowanie całego systemu z shell

### Już istnieje

```bash
subactor health
subactor status
subactor tickets
subactor get /api/projects/public-site-service-map
subactor get /api/projects/reconciliation
npm run verify:layers          # platform/scripts/verify-layered-system.mjs
npm run tickets:routes         # false_ready / unsupported URI
npm run doql:check
node scripts/run-public-site-service-map.mjs
```

### Nowe: reality-check

```bash
# z platform/
node scripts/reality-check-project.mjs contracts.subactor.com
node scripts/reality-check-project.mjs contracts.subactor.com --json

# CLI
subactor reality contracts.subactor.com
subactor reality contracts-subactor-com --json
```

Wyjście: public DNS, recon blockers, tickety, findings (`false_tls_human_boundary`, `plesk_domain_missing`, …), `recommended_next` URI.

### URI / REST — pełne testowanie z shell

| Potrzeba | Stan | Shell / REST |
|---|---|---|
| dig + ticket + recon + Plesk + attention | **jest** | `subactor reality <domain>` → `GET /api/projects/reality-check?domain=` |
| Plesk domains list (read-only) | **jest** | `subactor get '/api/connectors/plesk/domains?name=…'` → bridge `GET /plesk/domains` |
| URI process dry-run | **jest** | `subactor uri plesk://host/site/command/subdomain-ensure '{"domain":"…"}'` (domyślnie `apply=false`) |
| Attention policy | **jest** | w reality-check (classifyFounderAttention) + diagnostics |
| Service-map / recon | **jest** | `subactor get /api/projects/public-site-service-map` |

**Tak — da się testować cały łańcuch z clienta shell** (health → recon → reality → plesk observe → URI dry-run). Mutacje apply nadal tylko z ticketem + AQL + grantem.

```bash
export SUBACTOR_ADMIN_TOKEN=…
subactor health
subactor get /api/projects/reconciliation
subactor get '/api/projects/reality-check?domain=contracts.subactor.com'
subactor get '/api/connectors/plesk/domains?name=contracts.subactor.com'
subactor reality contracts.subactor.com --json
# dry-run (tworzy/używa process ticket; apply=false domyślnie dla ensure):
subactor uri 'plesk://host/site/command/subdomain-ensure' \
  '{"domain":"contracts.subactor.com","parent_domain":"subactor.com","apply":false}'
```

---

## Poprawki kodu (ta sesja)

1. **Attention** — ignore `blocked-by` / `blocked_by:PLF-*` (`blocked_by_dependency`).
2. **DNS remediation** — Plesk plane → `project-operator-bot`; multi-label → `subdomain-ensure` dry-run.
3. **TLS prompt** — gdy DNS unresolved, tekst: najpierw subdomena, nie SSL.
4. **Diagnosis** — `next_processes` startuje od subdomain-ensure.
5. **Shell** — `subactor reality` + `subactor uri` + `scripts/reality-check-project.mjs`.
6. **REST** — `GET /api/projects/reality-check`, `GET /api/connectors/plesk/domains`, bridge `GET /plesk/domains`.
7. **Lab Plesk 404** — lab→mock (osobna poprawka bridge).

---

## Jak czytać PLF-884 od teraz

| Ticket | Rola | Kto |
|---|---|---|
| **PLF-883** | Root: brak domeny w Plesk + DNS | **project-operator-bot** → `subdomain-ensure` + DNS |
| **PLF-884** | TLS po DNS | Founder tylko gdy DNS ready; inaczej `blocked_by:PLF-883` |
| **PLF-885** | App route | po DNS+TLS |

**Nie** zatwierdzaj ensure-ssl na PLF-884, dopóki:

```bash
dig +short contracts.subactor.com A @1.1.1.1   # = 217.160.250.222
# oraz Plesk domains zawiera contracts.subactor.com
subactor reality contracts.subactor.com         # findings puste / ok:true
```
