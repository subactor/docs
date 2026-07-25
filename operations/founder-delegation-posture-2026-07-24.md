---
{
  "schema": "subactor.doc/v1",
  "id": "docs.operations.founder-delegation-posture-2026-07-24",
  "version": 1,
  "status": "current",
  "updated": "2026-07-24"
}
---

# Postawa Foundera: zgoda i pomysły, nie kontrola ticketów

## Cel

Founder:

1. **TAK / NIE** na konkretny `plan_hash` (formularz),
2. **pomysły / planowanie** (intent, strategia, piloty biznesowe),
3. **sekrety / fakty prawne** (vault form, legal).

Founder **nie**:

- ręcznie prowadzi DNS, TLS, sync plików, ensure subdomeny,
- „babysituje” tickety remediacji,
- odblokowuje `master_mutation_gate` przez przeglądanie dziesiątek PLF-*.

## Podział odpowiedzialności

| Bloker | Właściciel | Rola Foundera |
|---|---|---|
| `dns_not_ready` | `project-operator-bot` | brak (chyba że sekrety Plesk) |
| `tls_not_ready` | `project-operator-bot` | TAK/NIE tylko na `ssl-ensure` apply |
| `application_route_not_ready` | `project-operator-bot` | TAK/NIE na dry-run `plan_hash` |
| `mutation_gate_disabled` | Founder (formularz) | jedna decyzja publikacji |
| `founder_publish_approval_required` | Founder | `approve_once` |
| `legal_documents_incomplete` | Founder | fakty / dokumenty |
| `plesk_system_profile_not_ready` | Founder | sekrety do vault |

## Bramki mutacji

Po zgodzie Foundera bot **wykonuje** plan, gdy:

- `AUTONOMY_MUTATIONS_ENABLED=1` (master gate),
- `PLESK_SYNC_APPLY=1` (sync plików httpdocs),
- podpisany apply-grant dla dokładnego `plan_hash` (gdzie wymagany).

Gdy master gate = 0, zatwierdzony plan parkował się na `administrator-bot` z
`master_mutation_gate_disabled` — stąd wrażenie „nic nie działa”.

## Shell (bez ręcznej kontroli ticketów)

```bash
subactor reality contracts.subactor.com
subactor get '/api/connectors/plesk/domains?name=contracts.subactor.com'
# dry-run (apply=false domyślnie):
subactor uri 'plesk://host/site/command/subdomain-ensure' \
  '{"domain":"contracts.subactor.com","parent_domain":"subactor.com"}'
```

## Implementacja (2026-07-24)

- `blocker-responsibility.mjs` — ops → bot
- `project-remediation.mjs` — TLS/route na `project-operator-bot`; boty startują `ready`
- po approval + `mutationsEnabled` → `apply_ready` na operator-bot (nie parking master gate)
- `founder-attention-policy` — ignore ops remediacji
- env: `AUTONOMY_MUTATIONS_ENABLED=1`, `PLESK_SYNC_APPLY=1`
