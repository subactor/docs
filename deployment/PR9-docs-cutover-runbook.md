# PR9 — docs.subactor.com Pages → Plesk cutover (runbook)

**Status:** preparation only — **do not** flip production DNS until every gate below is green.  
**Related:** ADR-002 (DNS SSOT), ADR-004 (publish DoD), ADR-005 (rollback), PR8 verify ladder.  
**Staging preference:** `docs-stage.subactor.com` (infra optional; origin Host / `curl --resolve` works without it).

## Honesty (2026-07-18 continuation #3 — subdomain rights + origin probe)

Observed **public** `docs.subactor.com` (read-only):

| Check | Result | Implication |
| --- | --- | --- |
| DNS CNAME | `subactor.github.io.` | Still on **GitHub Pages** |
| DNS A | `185.199.108–111.153` (Pages anycast) | ≠ Plesk origin |
| TLS SAN | `*.github.io` / `*.github.com` — **not** `docs.subactor.com` | Pages TLS is an **unhealthy** last_known_good for hostname DoD |
| `GET /__subactor_release.json` | TLS verify fails (SAN mismatch) | Public fingerprint DoD **cannot** pass until cutover + cert |

Observed **origin** via `curl --resolve docs.subactor.com:443:217.160.250.222` (Plesk IP = `dig A subactor.com`):

| Check | Result | Implication |
| --- | --- | --- |
| HTTPS `/` | **200** (probe `index.html`) | Dedicated subdomain docroot — **not** prototypowanie.pl |
| `/__subactor_release.json` | **200** probe marker | Origin fingerprint reachable via Host/`--resolve` |
| Origin TLS | self-signed `CN=Plesk` | G2 still red for public DoD |
| `docs-stage.subactor.com` public DNS | CNAME → `subactor.github.io` | Public staging still Pages; origin Host has Plesk default page |

**Subscription rights (XML API, customer `subactor_customer`):** `manage_subdomains=true`, `create_domains=true`, `max_subdom=-1`. Subdomains **created** 2026-07-18: `docs.subactor.com` (id 308), `docs-stage.subactor.com` (id 309); `www_root` = `/var/www/vhosts/subactor.com/docs[.stage].subactor.com`. Prior blocker was **connector gap** (no `site`/`subdomain` add URI), not a permission denial on the subscription.

Live SFTP (`plesk-sftp` / `subactor_ssh`) can write the docs docroot. **Public DNS unchanged** (still Pages).

**Cutover today: NO** — G2/G6 red; DNS HITL must not run.

## Gate status (2026-07-18 continuation #3)

| Gate | Status | Evidence / blocker |
| --- | --- | --- |
| **G0** Intent & ownership | **YELLOW** | Desired A filled (`217.160.250.222`); Pages CNAME saved as DNS emergency. Boundary HITL approval **not** recorded. |
| **G1** Origin content ready | **YELLOW** | Subdomains + dedicated docroots exist; SFTP probe uploaded `__subactor_release.json` (200 via `--resolve`). Formal release-upload→activate recipe + Cache-Control still pending. |
| **G2** Certificate plan | **RED** | Origin serves self-signed `CN=Plesk`; no agreed LE/DNS-01 SAN for `docs.subactor.com`. |
| **G3** DNS provider readiness | **YELLOW** | Desired state + reconcile stub exist; live mutate still HITL; TTL not lowered. |
| **G4** Verify ladder | **YELLOW** | Origin probe fingerprint OK via `--resolve`; public Pages failure still expected. |
| **G5** Rollback targets | **YELLOW** | Probe release on origin; DNS emergency = unhealthy Pages LKG. |
| **G6** Go / no-go | **RED** | Stop — do not mutate production DNS (cert + HITL). |

## Gates (all required before DNS mutate)

### G0 — Intent & ownership

- [ ] Cutover owned as **boundary-class** HITL (ADR-003) — not zero-touch reversible publish.
- [x] Desired DNS recorded in [`dns-desired-state.json`](./dns-desired-state.json) with real Plesk A/AAAA (no placeholder).
- [x] Previous desired (Pages → `subactor.github.io`) saved as emergency DNS rollback target (HITL only; **not** content LKG).

### G1 — Origin content ready (no public DNS required)

- [x] **Plesk subdomain** `docs.subactor.com` (+ optional `docs-stage`) under subscription `subactor.com` — **created 2026-07-18** via XML API (`subdomain.add`; ids 308/309). Separate docroots (not primary `/httpdocs`).
- [x] SFTP probe: `__subactor_release.json` + `index.html` on origin docroot; `--resolve` HTTPS 200 (2026-07-18).
- [ ] Formal release uploaded under Plesk release root (`releases/rel_…` + activate `current`) via connector recipe.
- [ ] Marker `Cache-Control: no-store` (or equivalent) on live responses.
- [ ] Prefer rehearsal content on **`docs-stage.subactor.com`** origin before production DNS.
- [x] Subscription permissions OK — **not** the blocker (`manage_subdomains` / `create_domains` true; unlimited `max_subdom`).
- [ ] **needs_human / connector:** wire safe `plesk://…` subdomain/site-add URI (today: ad-hoc XML + vault). Do not upload into primary prototypowanie.pl `httpdocs`.
- [x] Rebuild urirun-node with paramiko (SFTP) — **done 2026-07-18** (`production_publish_ready=true`; live methods sftp+ftp ok).

### G2 — Certificate plan

- [ ] Choose path: DNS-01 **before** cutover (preferred) **or** HTTP-01 LE immediately after DNS.
- [ ] Cert will include SAN `docs.subactor.com` (and staging hostname if used).
- [ ] Document who runs issuance (Plesk panel / ACME) — not LLM.

### G3 — DNS provider readiness

- [ ] Authoritative zone editable (provider API or panel).
- [ ] Intent stub [`dns-record-reconcile`](./dns-record-reconcile.urirun.json) reviewed; live mutate still HITL.
- [ ] TTL lowered ahead of cutover (e.g. 60–300s) with enough wait for caches.
- [ ] No residual CNAME to `*.github.io` after cutover.

### G4 — Verify ladder (PR8) against **desired** targets

Enable on recipe / CLI:

```text
plesk://host/site/command/publish-verify
  hostname=docs.subactor.com
  release_id=…
  artifact_sha256=…
  origin_ip=217.160.250.222          # pre-cutover
  dns_targets=[217.160.250.222]      # post-cutover
  verify_origin=true
  verify_public=true              # only after DNS+TLS green
  check_dns=true
  check_tls=true
```

- [ ] Pre-cutover: origin-only verify green; public DNS check still fails (Pages) — expected.
- [ ] Post-cutover: full ladder green → plan may reach `completed`.
- [ ] `200 + stale fingerprint` → `applied_unverified` → content rollback (`release-rollback`) or ticket — **never** `ok`/`completed`.

### G5 — Rollback targets (healthy)

| Layer | Healthy target | Unhealthy / notes |
| --- | --- | --- |
| Content | Previous Plesk release (`activate(previous)`) | GitHub Pages content ≠ Plesk release; no docs release on origin yet |
| DNS emergency | Prior CNAME → `subactor.github.io` (HITL) | Restores Pages; TLS SAN still wrong for `docs.subactor.com` until Pages/custom-domain cert fixed — **ops emergency only** |
| Staging | `docs-stage.subactor.com` on Plesk | Preferred rehearsal path — **not created** (public DNS still Pages) |

### G6 — Go / no-go

Cutover is allowed only when:

1. G1 origin fingerprint green,  
2. G2 cert path agreed (DNS-01 ready **or** immediate LE plan),  
3. G3 TTL↓ + provider access,  
4. Founder/admin HITL approval recorded,  
5. On-call can run content rollback + DNS emergency.

**If any gate is red → stop. Do not mutate production DNS.**

## Cutover sequence (when gates green)

1. Final origin verify (`--resolve`).
2. Issue/confirm cert (DNS-01) **or** prepare HTTP-01.
3. HITL: apply DNS A/AAAA (or ALIAS) → Plesk; remove Pages CNAME.
4. Wait TTL / check authoritative + public resolvers.
5. Confirm TLS SAN includes `docs.subactor.com`.
6. `publish-verify` with `verify_public=true` → fingerprint match.
7. Only then report publish / plan `completed`.
8. If verify fails → content `release-rollback` and/or DNS emergency HITL; status `rolled_back` / `applied_unverified`, never fake success.

## Automation hooks (prep)

| Hook | State |
| --- | --- |
| `plesk://…/publish-verify` | **PR8 done** (mocked + optional live) |
| `__subactor_release.json` on upload | **PR8 done** |
| Orchestrator `applied_unverified` | **PR8 done** |
| Desired DNS file | [`dns-desired-state.json`](./dns-desired-state.json) — A=`217.160.250.222` |
| `dns-record-reconcile` recipe stub | [`dns-record-reconcile.urirun.json`](./dns-record-reconcile.urirun.json) — **not** in intent-pack registry yet (avoids dual-run churn); register in PR9 when AQL + provider connector wired |
| Intent pack `dns-record-reconcile.v1.json` | Deferred until AQL model + namecheap/dns connector grant exist |

## Out of scope here

- Flipping production DNS without green gates.
- Claiming public `docs.subactor.com` is on Plesk.
- Uploading docs into primary `prototypowanie.pl` httpdocs as a substitute for the docs addon.

## Parallel work while PR9 blocked

See [`PR10-legacy-resolver-cleanup.md`](./PR10-legacy-resolver-cleanup.md) — reduce dual-run / legacy resolvers (pack-first already stable for docs/www).
