# Autonomia ops — status i pytania otwarte

**Data testów:** 2026-07-18  
**Zakres:** pełna autonomia operacyjna **bez self-evolution** (system nie przepisuje
samego siebie).  
**Kanoniczny runbook:** [`../autonomy-cli-runbook.md`](../autonomy-cli-runbook.md)  
**Plan publish:** [`../plans/docs-subactor-com-publish.md`](../plans/docs-subactor-com-publish.md)  
**Architektura intentów:** [`intent-orchestration-and-fallbacks.md`](./intent-orchestration-and-fallbacks.md)  
**Rekomendowane rozwiązanie:** [`autonomy-recommended-solution.md`](./autonomy-recommended-solution.md)  
**ADR (Faza 0):** [`adr/README.md`](./adr/README.md)  
**Roadmapa:** [`../plans/autonomy-implementation-roadmap.md`](../plans/autonomy-implementation-roadmap.md)  
**Baseline tego dokumentu:** commit `5894906` (snapshot diagnostyczny; nie mieszać z refaktorem orchestratora).

---

## 1. Aktualna sytuacja

Docelowa pętla:

```text
NL → intent → plan (ticket / recipe) → deploy (urirun) → verify → NL
```

### Co działa w pętli (dziś)

| Etap | Stan | Dowód |
| --- | --- | --- |
| NL → intent (frazy docs) | **Działa** | `subactor ask … --json` → pack SSOT / `docs-httpdocs-sync.pl.aql` (`source=intent-pack-registry` po unit 3; wcześniej `nlp-uri-phrase`) |
| Intent → ticket + plan | **Działa** | Ticket `PLF-353`, plan `proposed` (bez `--execute`) |
| Plan → AQL | **Działa** (founder) | `founder_admin_bypass` przy `SUBACTOR_ADMIN_TOKEN` |
| Dry-run deploy | **Działa** | Recipe `docs-httpdocs-sync`: methods + sync plan (~12 plików) |
| Live apply | **Nie działa end-to-end** | Brama `PLESK_SYNC_APPLY` OK; upload wcześniej timeout 30s (FTP) |
| Verify HTTPS publiczne | **Fail / mismatch** | `docs.subactor.com` → **GitHub Pages**, nie Plesk; TLS SAN bez `docs.subactor.com` |
| Zamknięcie pętli NL | **Częściowe** | Intent+plan OK; brak wiarygodnego „opublikowano i widać na HTTPS” |

### Mapa stacku (skrót)

| Warstwa | Rola | Uwaga |
| --- | --- | --- |
| Founder CLI `subactor` | Health, ask, tickets, plans | Control `:8091` healthy |
| Orchestrator `subactor-run` | Recipe / topo URI | Dry-run domyślnie |
| urirun-node | `plesk://…` HOW | SFTP: `paramiko_missing`; FTP: available |
| Vault / credentials | Lease do FTP/SFTP | Nie logowane; apply zależy od vault+transport |
| DNS / TLS | Prawda „co serwuje domenę” | docs ≠ subactor.com (Plesk `217.160.250.222`) |

**OpenRouter / LLM:** tylko intent (opcjonalnie). Upload jest deterministyczny
(connector + vault + `PLESK_SYNC_APPLY`).

**Poza zakresem tego dokumentu:** self-evolution (system przepisujący własny kod,
kontrakty lub politykę bez ludzkiego procesu change).

---

## 2. Wyniki testów (2026-07-18)

Środowisko: lokalny Docker platform (hr-control, planfile, urirun-node, …);
founder token załadowany z `platform/.env` (wartość **nie** zapisana tutaj).

| # | Test | Wynik | Fakty (bez sekretów) |
| --- | --- | --- | --- |
| 1 | `subactor health` | **PASS** | `{"ok":true,"service":"organization-control"}` |
| 2 | `subactor ask "…" --json` | **PASS** | `source=intent-pack-registry` (unit 3) lub legacy `nlp-uri-phrase`, model `docs-httpdocs-sync.pl.aql`, situation `docs` → `docs.subactor.com` |
| 3 | `subactor ask "…"` (propose) | **PASS** | Ticket `PLF-353`, plan `plan_mrpyf6eq_53aae9b275`, status `proposed` |
| 4 | Recipe dry-run `docs-httpdocs-sync.urirun.json` | **PASS** | Methods OK; dry-run `files_planned=12`; recommended transport `ftp` |
| 5 | Recipe `--execute` **bez** `PLESK_SYNC_APPLY` | **PASS (brama)** | Apply step: `plesk_sync_apply_required` — upload zablokowany zgodnie z polityką |
| 6 | Live apply z `PLESK_SYNC_APPLY=1` | **FAIL** (wcześniej dziś; nie powtarzane jako sukces) | Timeout ~30s na FTP upload; SFTP niedostępne (`paramiko_missing`) |
| 7 | `https://docs.subactor.com/` (strict TLS) | **FAIL** | curl 60: cert `CN=*.github.io`, brak SAN dla `docs.subactor.com` |
| 8 | Treść / kto serwuje (curl `-k`) | **INFO** | `server: GitHub.com`, Jekyll; CNAME → `subactor.github.io` / `185.199.*` |
| 9 | Porównanie `subactor.com` | **INFO** | A → `217.160.250.222` (Plesk); docs **nie** wskazuje Pleska |
| 10 | Kontener urirun: `PLESK_SYNC_APPLY` / paramiko | **INFO** | Apply env **unset**; `paramiko` **no** |

**Wniosek:** ścieżka **NL → intent → ticket/plan → dry-run** jest żywa.
Ścieżka **apply → publiczne HTTPS na Plesku** nie jest zamknięta (DNS/TLS +
transport/timeout). **Nie twierdzimy o udanym live publish.**

---

## 3. Problemy ops

| Problem | Objaw | Skutek dla autonomii |
| --- | --- | --- |
| DNS docs → GitHub Pages | CNAME `subactor.github.io` | Sync do Plesk httpdocs nie zmienia tego, co widzi świat |
| TLS SAN mismatch | Cert `*.github.io` vs hostname `docs.subactor.com` | Strict verify / automatyczny health check pada |
| Brak paramiko w urirun-node | `sftp.available=false`, `paramiko_missing` | Tylko FTP; wolniejszy/mniej niezawodny upload |
| Timeout apply (~30s) | Live FTP sync fail | Recipe `--execute` z bramą i tak nie domyka publish |
| `PLESK_SYNC_APPLY` unset | Domyślnie bezpiecznie | Founder musi świadomie ustawić bramę na nodzie |
| Docroot `/httpdocs` | Recipe targetuje primary httpdocs | Ryzyko nadpisania złego vhostu; preferowany addon `/docs.subactor.com` (ops) |
| Domain/subscription preflight | Brak automatycznego kroku „domena + TLS na Plesku” | NL obiecuje publish bez gwarancji infrastruktury |
| Vault ensure | Często ręczne | Autonomia urywa się na brakującym `plesk-sftp` / FTP |
| urirun-lan-gateway unhealthy | Docker status | Potencjalny szum ops / ścieżki LAN |
| Secrets ownership | Tokeny w `.env`, vault w browser-agent | Brak jasnego „kto rotuje / kto lease’uje z CLI” |

---

## 4. Problemy produktowe / platformowe (nie ops)

Krótko — szczegóły w
[`intent-orchestration-and-fallbacks.md`](./intent-orchestration-and-fallbacks.md):

| Problem | Skutek |
| --- | --- |
| Intent wiring zduplikowany (frazy, AQL, step-catalog, recipe, Planfile) | Drift; drogi cost nowego celu |
| Brak intent pack SSOT | Każdy cel = N plików ręcznie |
| Orchestrator NL stub ≠ pełna multi-step recipe | `--nl` nie zastępuje ticketu z `uri_processes` |
| Fail-fast `runTask` bez `optional` / `on_fail` | Preflight zabija cały plan |
| Kontrakty autonomii w API to głównie TestQL onboarding | Brak produkcyjnego kontraktu „docs publish” |
| Multi-goal NL (treść + publish) niepełne | Intent trafia w sync; generacja treści nie jest w tym samym łańcuchu |
| Misroute LLM bez trafienia frazy | Historycznie modele onboardingowe — łata phrase map, nie SSOT |

---

## 5. Pytania do rozstrzygnięcia

Checklist decyzji pod **pełną autonomię poza self-evolution**.  
*(Self-evolution: out of scope — jawnie wykluczone.)*

### Governance / apply

- [ ] **Kto zatwierdza apply?** Founder zawsze bypass (`SUBACTOR_ADMIN_TOKEN` + `*`)
      vs polityka kontraktu (bot tylko dry-run; apply po human / po nazwanym kontrakcie)?  
      **Rekomendacja:** kill switch + podpisany apply grant związany z `plan_hash`/artefaktem/targetem; founder bypass nie zastępuje grantu w docelowym modelu ([ADR-003](./adr/003-approval-hitl-model.md)).
- [ ] **Dry-run zawsze przed apply?** Czy obowiązkowy w recipe policy, czy wystarczy
      brama `PLESK_SYNC_APPLY`?  
      **Rekomendacja:** obowiązkowy dry-run → immutable manifest; `PLESK_SYNC_APPLY` / `AUTONOMY_MUTATIONS_ENABLED` tylko jako kill switch, nie jedyna autoryzacja.
- [ ] **Human-in-the-loop when?** Tylko ensure credentials / DNS / TLS, czy też
      każdy live mutate poza allowlistą?  
      **Rekomendacja:** HITL dla boundary (domena, DNS, credential) i governance (nowy pack/polityka); reversible mutate (kolejny release) zero-touch po dry-run ([ADR-003](./adr/003-approval-hitl-model.md)).

### Prawda domeny i weryfikacja

- [ ] **Gdzie żyje prawda DNS/domen?** Repo CNAME, Plesk API, zewnętrzny DNS panel —
      jeden SSOT + preflight URI przed obietnicą NL?  
      **Rekomendacja:** repo = desired state; provider = observed; connector DNS = reconcile; Plesk nie jest SSOT DNS ([ADR-002](./adr/002-dns-ssot.md)).
- [ ] **Monitoring / verify obowiązkowe?** Czy plan bez HTTPS/DNS check może być
      `ok: true`, czy verify jest częścią Definition of Done autonomii?  
      **Rekomendacja:** verify obowiązkowy; `upload OK + verify FAIL` = `applied_unverified` ≠ sukces ([ADR-004](./adr/004-publish-definition-of-done.md)).
- [ ] **GitHub Pages vs Plesk:** docs zostaje na Pages, czy migracja DNS→Plesk
      (wzorzec `subactor.com`)?  
      **Rekomendacja:** `docs.subactor.com → Plesk`; jeśli Pages — usunąć intent docs→Plesk z procesu publicznego ([ADR-002](./adr/002-dns-ssot.md)).

### Zdolności i connectorzy

- [ ] **Jakie capability muszą być w connectorach zanim NL może obiecać wynik?**
      Minimum: transport (SFTP lub FTP), vault lease, allowlist source, domain
      exists, TLS OK, apply gate, post-verify.  
      **Rekomendacja:** pack deklaruje `required_capabilities`; preflight musi być green przed obietnicą sukcesu; minimum jak powyżej + release activate/rollback.
- [ ] **Paramiko / SFTP w obrazie urirun-node** — wymagane przed „autonomicznym
      publish”, czy FTP + wyższy timeout wystarczy?  
      **Rekomendacja:** paramiko w obrazie; brak SFTP blokuje readiness produkcyjnego publish; FTP tylko jako fallback konektora, nie jedyna ścieżka.
- [ ] **Timeout / retries:** stałe 30s vs per-URI budget; kto ustawia?  
      **Rekomendacja:** policy connectora/recipe (np. connect 15s, op 120s, budget 180s, 3× backoff 1/3/9); nie LLM.

### Secrets / vault

- [ ] **Ownership sekretów?** Kto tworzy, rotuje, lease’uje z CLI/recipe bez wklejania
      do ticketów?  
      **Rekomendacja:** człowiek tworzy/rotuje; vault = SSOT; recipe = `credential_ref`; runtime = krótki lease ([ADR-006](./adr/006-secrets-ownership.md)).
- [ ] **Ensure SFTP → vault** zawsze pierwszym krokiem, czy opcjonalny preflight?  
      **Rekomendacja:** brak credential → `needs_human` + ticket bootstrap; bez publikacji i bez pytania LLM o hasło.

### Scope produktu

- [ ] **Scope „dowolne zadanie” vs katalog intent packs?** Autonomia = zamknięty
      katalog nazwanych celów, czy free-form LLM z eskalacją?  
      **Rekomendacja:** kontrolowany katalog intent packów; LLM wybiera pack + sloty, nie URI ([ADR-001](./adr/001-autonomy-scope.md)).
- [ ] **Rollback / failure semantics?** Halt + ticket; partial upload rollback;
      `on_fail: continue|ticket|halt` w recipe policy?  
      **Rekomendacja:** `on_fail: halt|continue|ticket|rollback`; release-based rollback + stany planu bogatsze niż boolean ([ADR-005](./adr/005-rollback.md)).

### Poza zakresem (nie rozstrzygamy tu)

- Self-evolution: auto-zmiana kodu, kontraktów AQL, allowlist, lub polityki
  autonomii przez samego agenta — **wykluczone**.

---

## 6. Definition of done — „pełna autonomia (bez ewolucji)”

Mierzalne kryteria. Wszystkie muszą być prawdziwe dla reprezentatywnego celu
(np. docs publish) **oraz** dla nowego celu z katalogu intent packs bez ręcznego
„sklejania” N plików poza packiem.

| # | Kryterium | Jak zmierzyć |
| --- | --- | --- |
| D1 | NL founder → poprawny named intent bez OpenRouter (phrase) lub z LLM tylko przy braku frazy | `subactor ask --json` → oczekiwany `model_name` |
| D2 | Intent → ticket z pełnym `uri_processes` (nie pojedynczy URI stub) | Planfile / plan proposed z multi-step |
| D3 | Dry-run zawsze przed live mutate | Recipe/policy; assert w TestQL |
| D4 | Live apply tylko przy jawnej bramie + kontrakcie/aktorze | Bez `PLESK_SYNC_APPLY` → `plesk_sync_apply_required`; z bramą → upload OK |
| D5 | Wymagane connector capability green przed obietnicą sukcesu | Preflight: methods, vault, domain, TLS |
| D6 | Publiczny verify po apply | `curl -fsS https://<domain>/` HTTP 200 + oczekiwany fingerprint treści |
| D7 | DNS/TLS zgodne z celem publish | Dig + cert SAN zawierają hostname |
| D8 | Failure → ticket / escalate, nie ciche `ok` | Status planu + Planfile history |
| D9 | Zero sekretów w ticketach, recipe, logach NL | Audit / grepowalne ścieżki |
| D10 | Nowy cel z intent pack bez driftu N kopii | Jeden pack → phrases + allow + recipe ref |
| D11 | Self-evolution wyłączone | Brak URI/procesu „rewrite self”; change tylko przez ludzki change process |

**Minimum dla docs.subactor.com:** D1–D9 spełnione przy DNS→Plesk (lub świadomej
decyzji „zostajemy na GitHub Pages” i wtedy sync Plesk **nie** jest częścią DoD
publicznego HTTPS).

---

## 7. Powiązane dokumenty

| Doc | Rola |
| --- | --- |
| [`../autonomy-cli-runbook.md`](../autonomy-cli-runbook.md) | CLI / pętla NL→URI |
| [`../plans/docs-subactor-com-publish.md`](../plans/docs-subactor-com-publish.md) | Plan + wcześniejsze wyniki testów publish |
| [`../deployment/PLESK.md`](../deployment/PLESK.md) | Ops path docs → httpdocs |
| [`../deployment/docs-httpdocs-sync.urirun.json`](../deployment/docs-httpdocs-sync.urirun.json) | Recipe dry-run/apply |
| [`intent-orchestration-and-fallbacks.md`](./intent-orchestration-and-fallbacks.md) | Intent packs, policy, fallbacki |
| [`autonomy-recommended-solution.md`](./autonomy-recommended-solution.md) | Kanoniczna rekomendacja autonomii |
| [`adr/README.md`](./adr/README.md) | ADR Phase 0 (do akceptacji) |
| [`../plans/autonomy-implementation-roadmap.md`](../plans/autonomy-implementation-roadmap.md) | Fazy 0–8 + kolejność zmian |
| [`../plans/intent-capability-fallbacks.md`](../plans/intent-capability-fallbacks.md) | Krótka nota planowa |
| [`../../platform/docs/URI_PROCESS_AUTONOMY.md`](../../platform/docs/URI_PROCESS_AUTONOMY.md) | AQL / OQL / URI |
| [`../../platform/docs/AUTONOMY_CONTRACTS.md`](../../platform/docs/AUTONOMY_CONTRACTS.md) | Kontrakty autonomii |
| [`../../www/deployment/PLESK.md`](../../www/deployment/PLESK.md) | Wzorzec www → subactor.com (działający) |

---

*Wygenerowano po praktycznych testach CLI/recipe/DNS (2026-07-18). Bez sekretów.*
