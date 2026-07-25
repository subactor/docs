---
{
  "schema": "subactor.doc/v1",
  "id": "docs.operations.plesk-catalog-ticket-audit-2026-07-24",
  "version": 1,
  "status": "current",
  "updated": "2026-07-24"
}
---

# Audit: katalog Plesk vs tickety + wykonanie PLF-883

## 1. Czy queue odpalił kroki PLF-883?

**Nie przez URI Process queue.**

| Fakt | Stan (2026-07-24 ~20:27 UTC) |
|---|---|
| PLF-883 `status` | `done` |
| `uri_processes[*].status` | wszystkie `pending` |
| Notatka | `CLI 2026-07-24: subdomain contracts.subactor.com id=315; DNS A 217.160.250.222 public ready` |
| PLF-884 | `done`, notatka CLI o LE TLS |
| Live dig | `217.160.250.222` |
| Live Plesk domains | `contracts.subactor.com` id=315 |
| Live cert (SNI) | CN/SAN = `contracts.subactor.com`, ważne do 2026-10-22 |
| `subactor reality` | **ok: true**, findings=0 |

**Wniosek:** efekt produkcyjny (domena + DNS + cert) powstał **poza** autonomous queue (CLI / ręczna ścieżka). Queue **nie** oznaczył kroków URI jako completed — ticket `done` z `pending` processami to rozjazd receipts.

### Dlaczego queue nie jechał

- `delegation/manager.queue_consumer`: `error=planfile_unavailable`, `considered=0`, `results=[]`.
- Control → Planfile (`http://planfile:8000`) **timeoutuje** przy listach/health pod obciążeniem.
- Bez stabilnego Planfile consumer nie bierze ticketów `ready` do `execute`.

---

## 2. Audit: catalog vs URI w otwartych ticketach

### Katalog Plesk (19 capability → 22 URI)

Zamknięty zbiór w `connector-capabilities/catalog.v1.json`:

- sync, subdomain-ensure, domain-ensure, ssl-ensure
- DNS: authority, records, replace, reconcile, propagation, preflight
- doctor, subscription capabilities, publish-verify, reverse-proxy
- transports SFTP/FTP, extensions (profiled)

### Otwarte tickety z `plesk://`

| URI w ticketach | W katalogu? |
|---|---|
| `…/site/command/sync` | TAK (14 ticketów) |
| `…/site/command/subdomain-ensure` | TAK |
| `…/site/command/ssl-ensure` | TAK |
| `…/dns/*` (authority, replace, propagation) | TAK |
| `…/doctor`, `subscription`, `extensions/catalog` | TAK |
| `…/site/command/reverse-proxy-ensure` | TAK |
| `…/site/command/publish-verify` | TAK |
| **`plesk://host/site/query/methods`** | **NIE** (PLF-504, PLF-508) |

**Pokrycie:** 11/12 unikalnych `plesk://` z otwartych ticketów jest w katalogu.  
Jedyna luka katalogowa w użyciu: **`site/query/methods`**.

### Bramy `human_approval=true` (wciąż pending)

Operacyjne (powinny iść na bot + 1× consent, nie ręczny DNS za każdym razem):

- `reconcile-dns` → `dns/command/replace` (PLF-1493) — **human=true** → `human_boundary` na bot queue
- `ensure-ssl` apply (PLF-1494, PLF-592)
- `publish` / `site-sync-apply` (wiele projektów)
- `approve` task:// (PLF-885, 1053, 1056, 894, 913, 914…)

Biznesowe (zostawić Founderowi):

- pilot terms, recruitment, social onboarding, incident-response

---

## 3. contracts.subactor.com — łańcuch vs queue

```text
[OK live]  subdomain w Plesk (id=315)
[OK live]  public DNS A = origin
[OK live]  LE cert SAN = contracts.subactor.com
[OK]       reality-check findings=[]
[FAIL path] queue URI steps nie odpalone / nie zreceiptowane
[FAIL path] PLF-885 apply_ready + approval, ale brak kroku apply-sync w processach
             (dry-run → approve → verify, bez publish apply)
[FAIL try]  POST processes/run sync dry-run → plesk_site_source_dir_invalid
             (zły source_dir w ręcznym wywołaniu audytu)
[FAIL dep]  Planfile latency → autonomous queue idle
```

Nowe tickety po recon: **PLF-1493** (DNS, czek na human reconcile-dns), **PLF-1494** (TLS blocked-by 1493) — mimo że live DNS/TLS już zielone → **fałszywa remediacja** vs live observation (reality-check to widzi, recon jeszcze nie domknął blockerów).

---

## 4. Czy LLM mógłby to domknąć z katalogu?

| Warstwa | Ocena |
|---|---|
| URI w ticketach remediacji | **Wystarczająco w katalogu** (prawie 100%) |
| Payload / source_dir / remote_path | **Często niedookreślone** — muszą z manifestu projektu |
| Wykonanie | **Bot + Planfile + queue**, nie LLM |
| Blokada dziś | Planfile timeout + human_approval na DNS replace + brak apply w processie po approval |

**Schema katalogu wystarcza botowi na łańcuch domena→DNS→TLS→sync**, o ile:

1. Planfile jest responsywny,
2. botowe kroki startują `ready` bez fałszywego `human_boundary`,
3. po `publication_approval` jest jawny krok `sync apply=true`,
4. payload bierze `source_dir` z manifestu.

LLM nie jest potrzebny do tego łańcucha i nie powinien inventować URI spoza katalogu.

---

## 5. Rekomendowane naprawy (kolejność)

1. **Stabilność Planfile** (timeouty list ticketów) — bez tego queue = 0.
2. **Po live green (reality-check ok)** — recon nie tworzy PLF-1493/1494; domyka stare remediacje.
3. **DNS replace:** dry-run bez human; apply tylko po jednorazowym plan_hash / mutations gate (nie każdy reconcile).
4. **Dodać do katalogu** `plesk://host/site/query/methods` (używane w PLF-504/508).
5. **PLF-885 process:** wstawić `apply-sync` po `approve` gdy approval już jest.
6. **Receipts:** CLI/ops success → oznacz URI steps completed lub cancel duplikaty.

---

## 6. Shell weryfikacja (teraz)

```bash
subactor reality contracts.subactor.com          # ok: true
subactor get '/api/connectors/plesk/domains?name=contracts.subactor.com'
# cert:
echo | openssl s_client -connect 217.160.250.222:443 -servername contracts.subactor.com 2>/dev/null \
  | openssl x509 -noout -subject -dates
```
