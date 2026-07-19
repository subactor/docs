# Raport stanu platformy Subactor — publikacja Plesk

**Data:** 2026-07-19 · **Zakres:** ścieżka `subactor ask` → intent pack → ticket → plan → dry-run → apply (Plesk), stan po promocji generycznego packa `site-publish` (SELFDEV-062).

> **Aktualizacja 2026-07-19 (po refaktoryzacji):** wykonano punkty **2.1** (commit promocji), **2.2** (tokenowy matcher fraz), **2.5** (cały stack z repo, zero mountów `/tmp`), **3.1** (podział `registry.mjs` na 4 moduły) i **3.3** (wspólny `config-paths.mjs`). Szczegóły w treści — wykonane pozycje oznaczone ✅. Testy po refaktoryzacji: intent-packs 30/30, control 113 (0 fail), modele AQL OK; e2e `ask` z identycznymi `plan_hash` (www `1418bcbd…`, identity `3b19a968…`). Fraza z wtrąconym słowem („opublikuj stronę **identity** na plesk") rozwiązuje się teraz poprawnie do packa `site-publish`.
>
> **Aktualizacja 2026-07-19 (wieczór):** domknięto prace wokół digital twin / ciągłości decyzyjnej — subject-bound scope `digital-twin:self:read`, synchronizację env-contract (`npm run env:sync`) i plan „constitutional continuity". Szczegóły w sekcji **6**.

Powiązane dokumenty:

| Dokument | Zakres |
| --- | --- |
| [`autonomy-cli-runbook.md`](../autonomy-cli-runbook.md) | Runbook CLI: NL goal → URI Process |
| [`koru-subactor-autonomy-assessment-2026-07-18.md`](koru-subactor-autonomy-assessment-2026-07-18.md) | Ocena autonomii (dzień wcześniej) |
| `platform/docs/GITHUB_PLESK_URI_PROCESSES.md` | Przepisy GitHub + Plesk |

---

## 1. Co działa poprawnie (zweryfikowane e2e)

Wszystkie testy wykonano wyłącznie przez `subactor ask`, na `PLESK_MODE=mock` (bezpieczne apply do mocka; ścieżka produkcyjna wymaga świadomej decyzji foundera).

| Obszar | Dowód | Źródło |
| --- | --- | --- |
| Control plane `:8091`, bridge, urirun-node | 15/15 usług zdrowych, 0 błędów krytycznych | `platform/docker-compose.yml` |
| Intent → ticket → plan → dry-run (4 domeny) | www 23 pliki · docs 82 · docs-stage 82 · logo 73; deterministyczne packi, koszt LLM $0, preflight OK | `platform/config/intent-packs/*.v1.json` |
| Apply z pełnym governance | podpisany apply-grant związany z `plan_hash` → mutate lease TTL 900 s → upload SFTP z manifestem SHA-256 → lease cleared | `platform/components/runtime/src/apply-grant.mjs`, `platform/bin/subactor` |
| **Nowe domeny — generyczny pack `site-publish`** (aktywowany 2026-07-19) | `--field source_ref=workspace:identity` → `identity.subactor.com`, dry-run 22 pliki → docroot `/identity.subactor.com`, mock apply OK | `platform/config/intent-packs/site-publish.v1.json`, `platform/components/core/services/control/src/site-publish-resolver.mjs` |
| Testy | intent-packs 30/30 · control 113 (0 fail) · modele AQL OK · regresja 4 domen z identycznymi `plan_hash` | `platform/test/intent-packs.test.mjs`, `platform/components/core/services/control/tests/` |
| Preflight produkcyjny | vault kompletny (`plesk-runtime`, `plesk-subscription`, `plesk-ftp/sftp`, `deploy-ftp/sftp`), HTTPS 200 na docs/logo/www/identity.subactor.com | `platform/bin/subactor-live-publish.sh` |

Przykładowe wywołania (działające):

```bash
subactor ask "zsynchronizuj www do httpdocs subactor.com" --execute --yes     # dry-run
subactor ask "zaktualizuj zawartość folderu www i opublikuj na subactor.com" --apply --yes
subactor ask "opublikuj stronę na plesk" --field source_ref=workspace:identity --execute --yes
```

Co było potrzebne do promocji `site-publish` (wykonane):

- zdjęcie flagi `shadow` z packa — `platform/config/intent-packs/site-publish.v1.json`
- model AQL — `platform/components/contracts/models/site-publish.pl.aql`
- recipe — `platform/config/recipes/site-publish.urirun.json`
- scenariusz wykonywalny — `platform/config/scenarios.json`
- hook deterministycznego rozwiązania `source_ref → source_dir + domain + remote_path` przed walidacją AQL — `platform/components/core/services/control/src/routes/plans.mjs` (`handleProposeFromIntent` → `buildSitePublishSituation`)
- mounty zasobów workspace (ro) do hr-control i `identity` do urirun-node — `platform/docker-compose.yml`
- `PLESK_SYNC_ALLOWED_SOURCES=/home/tom/github/subactor` na urirun-node (tryb prefiksowy zamiast basename `{www,docs,logo}`) — `platform/docker-compose.yml`, logika: `urirun-connectors/urirun-connector-plesk/urirun_connector_plesk/core.py` (`_source_allowed`)

---

## 2. Co wymaga poprawy

### 2.1 ✅ Nieskomitowane zmiany promocji site-publish — WYKONANE

Zmiany skomitowane w submodułach i platformie (commity „refactoring" + `chore: pin agents submodule at 712bc45`). Submoduły `components/core` i `components/contracts` zakotwiczone na branchu `main` (były detached HEAD — ryzyko utraty commitów). Pin agents w `platform/test/intent-packs.test.mjs` zaktualizowany do `712bc455…`.

### 2.2 ✅ Dopasowanie fraz jest substring-owe — WYKONANE (tier tokenowy)

Nowy `platform/config/intent-packs/phrase-matcher.mjs`: tier 1 to historyczny exact/substring (wyczerpywany w całości dla wszystkich packów — zero zmian rankingu istniejących dopasowań), tier 2 to podciąg tokenów z ograniczonymi wtrąceniami (≤2 tokeny między kolejnymi słowami frazy). „opublikuj stronę **identity** na plesk" trafia teraz w pack `site-publish` (potwierdzone e2e, `plan_hash` identyczny z czystą frazą). `resolveNlpUriFromPacks` celowo pozostał substring-only (bezpośrednia mapa fraza→URI pozostaje konserwatywna).

*Pozostało (opcjonalnie):* wpięcie ekstraktora slotów `site-intent-extractor.mjs` w ścieżkę `ask`, żeby `source_ref` wyciągał się z NL bez `--field`.

### 2.3 Rosnący backlog eskalacji

„Do uwagi człowieka": 105 → 135 w ciągu jednej sesji; 436 ticketów, w tym dziesiątki duplikatów `Przygotuj dostęp zdalny: Anna Nowak [pending]` i auto-zakładane SELFDEV przy każdym nieudanym ask (`SELFDEV-005/-011/-062/-073…`).

**Propozycja:** deduplikacja ticketów po `(typ, hash sytuacji)` przy tworzeniu oraz TTL/auto-close dla SELFDEV, których przyczyna zniknęła. Miejsca: `platform/components/core/services/control/src/routes/plans.mjs` (ścieżka tworzenia), planfile.

### 2.4 Vault zaśmiecony wpisami testowymi

~90 wpisów `plesk-mailbox-*@example.test` obok produkcyjnych credentiali. Utrudnia audyt, zwiększa powierzchnię pomyłki.

**Propozycja:** flaga/prefiks `test:` przy tworzeniu + komenda czyszcząca wpisy z originem `example.test` (browser-agent vault API).

### 2.5 ✅ Uruchamianie stacka ze snapshotu `/tmp` — WYKONANE

`docker compose --env-file .env up -d` z katalogu `platform/` przejęło cały projekt: `docker compose ls` pokazuje wyłącznie ścieżki repo, zero bind-mountów `/tmp` na wszystkich kontenerach, 17/17 usług up (15 z healthcheckiem — healthy). Smoke test `ask` po przejęciu: `plan_hash` bez zmian. Klasa awarii „ENOENT po czyszczeniu `/tmp`" zamknięta.

### 2.6 Kontenery spoza platformy w restart-loopie

`service-manager-pilot-backend`, `service-id-pilot-backend` (`~/github/maskservice/c2004`), `proxym` (`~/github/semcod/proxym`). Poza zakresem subactora, ale szumią w monitoringu hosta.

### 2.7 Preflight live-publish oparty o `pgrep`

`platform/bin/subactor-live-publish.sh` sprawdza `pgrep -f "urirun node serve"` — działa przypadkiem (procesy kontenera widoczne z hosta). Pewniejszy: `curl -fsS http://127.0.0.1:18765/health`.

---

## 3. Co wymaga modularyzacji

### 3.1 ✅ `platform/config/intent-packs/registry.mjs` — WYKONANE (podział na 4 moduły)

Zrealizowany podział (registry.mjs pozostał fasadą z niezmienionym publicznym API — wszyscy importerzy działają bez zmian):

| Moduł | Odpowiedzialność |
| --- | --- |
| `pack-loader.mjs` | load + shape assert + shadow gating |
| `phrase-matcher.mjs` | dopasowanie NL: tier substring + tier tokenowy (patrz 2.2) |
| `docroot-rules.mjs` | branche domain/remote_path wyniesione verbatim (parytet `plan_hash`) |
| `derived-artifacts.mjs` | nlp-uri entries, render YAML, LLM slice, compare recipe/step-catalog |

*Pozostało (TODO w `docroot-rules.mjs`):* przenieść reguły per-domain do danych packów i domykać generyczną `docrootConvention` z `site-publish-resolver.mjs` — wtedy żadna nazwa domeny nie będzie hardkodowana w kodzie.

### 3.2 `platform/bin/subactor` (764 linie bash + inline Python)

Cała orkiestracja ask→ticket→plan→grant→apply w jednym skrypcie; nietestowalne, logika lifecycle zduplikowana z orchestratorem.

**Propozycja:** logika do `orchestrator/` (Node, testy jednostkowe), bash jako cienki wrapper na REST.

### 3.3 ✅ Zduplikowana logika ścieżek konfiguracji — WYKONANE

Nowy `platform/components/core/services/control/src/config-paths.mjs` (`configFileCandidates` / `resolveConfigFile` / `packsDirCandidates`) zastąpił 4 kopie łańcucha „env → CONTROL_CONFIG_DIR → umbrella → submodule pin" w: `docs-sync-intent.mjs`, `www-sync-intent.mjs`, `site-resource-resolver.mjs`, `publish-autonomy.mjs`. Testy control 113 (0 fail), e2e bez zmian `plan_hash`.

### 3.4 Legacy intent-moduły vs packi

`docs-sync-intent.mjs` i `www-sync-intent.mjs` duplikują logikę packów; dual-run `shadow` (PR10) miał je wygasić. Utrzymywanie obu = dryf (realny przypadek: stash „docs-sync-intent stage bind" z 2026-07-18 vs HEAD).

**Propozycja:** po okresie shadow bez mismatchy → `INTENT_PACK_DUAL_RUN=off` i usunięcie obu plików.

### 3.5 Podwójny `agents/nlp-uri-phrases.yaml`

Generat istnieje w `agents/nlp-uri-phrases.yaml` (umbrella) i w pinowanym klonie `platform/components/agents/nlp-uri-phrases.yaml`; przy promocji packa synchronizowany ręcznie (`cp`).

**Propozycja:** `platform/scripts/sync-intent-pack-derived.mjs --write` pisze do obu lokalizacji, albo `components/agents` czyta generat z config packów w runtime.

### 3.6 Generyczne id kroków w step-catalog

`create_site_publish_ticket` reużywa id `www-httpdocs-*` (`platform/config/step-catalog.json`), co myli w logach („www" przy publikacji identity).

**Propozycja:** neutralne id `site-methods` / `site-sync-dry-run` / `site-sync-apply` w katalogu + w `platform/config/recipes/site-publish.urirun.json` (komparator `compareRecipeToStepCatalog` wymaga tylko spójności obu miejsc — zmiana lokalna).

### 3.7 Zduplikowany testkit

`testkit/tests/testql/` (root umbrelli) i `platform/components/testkit/tests/testql/` są bit-w-bit identyczne (zweryfikowane na `panel-gui.testql.toon.yaml`).

**Propozycja:** zostawić `platform/components/testkit`, root-owy katalog → symlink albo usunięcie.

---

## 4. Rekomendowana kolejność

1. ~~**Commit promocji site-publish** (2.1)~~ ✅ wykonane 2026-07-19
2. ~~**Przejęcie całego stacka z repo** (2.5)~~ ✅ wykonane 2026-07-19
3. ~~**`phrase-matcher`** (2.2 + 3.1)~~ ✅ wykonane 2026-07-19 (bez wpięcia ekstraktora slotów — opcjonalne follow-up)
4. **Dedup ticketów / SELFDEV TTL** (2.3) — zatrzymuje puchnięcie backlogu. **← następny krok**
5. Pozostałe: 2.4 (vault cleanup), 2.6 (kontenery obce), 2.7 (`pgrep`→HTTP), 3.2 (CLI→orchestrator), 3.4 (wygaszenie legacy intents), 3.5 (podwójny YAML), 3.6 (neutralne id kroków), 3.7 (zduplikowany testkit) — osobne, małe PR-y.

## 5. Wykonane commity (2026-07-19)

| Repo | Commit | Zakres |
| --- | --- | --- |
| `platform` | „refactoring" (użytkownik) + `chore: pin agents submodule at 712bc45` | promocja site-publish, pin agents |
| `platform` | `refactor(intent-packs): split registry.mjs into focused modules + token-tier phrase matching` (`eb70758`) | 3.1 + 2.2 |
| `platform/components/core` | `refactor(control): extract shared config-paths.mjs (4 copies -> 1)` (`1d6133d`) + bump pinu w platform (`7e1b2b0`) | 3.3 |
| `platform/components/contracts` | `chore: ignore __pycache__` (`96b7245`) | porządek |

## 6. Prace domknięte po refaktoryzacji (2026-07-19, wieczór)

Zmiany niezwiązane bezpośrednio ze ścieżką publikacji Plesk, ale dotykające tej samej platformy:

### 6.1 Subject-bound digital twin

Scope `digital-twin:self:read` jest dostępny w presetach tokenów i E2E. Endpoint `GET /api/digital-twin/self` wiąże widok z principalem tokenu, sprawdza AQL `twin://actor/digital-twin/query/self`, zwraca wyłącznie projekcję read-only i nie udostępnia lookupu cross-subject.

Test live potwierdził tę izolację dla 8/8 zarejestrowanych aktorów. Implementacja: Core `075fc0f`, kontrakty `a47e34b`, testkit `4959261`, pin platformy `fb741ec`.

### 6.2 Tooling env-contract

`platform/scripts/sync-env-contract.mjs` i npm script **`env:sync`** wciągają aktualny `.env.example` jako `template` do `platform/config/env-contract.json`. `.env.example` jest SSOT, a kontrakt jest generowany i kontrolowany przez test driftu.

Konfiguracja Foundera nie zakłada już nazwy kontraktu w kodzie Organization Core: `FOUNDER_PRINCIPAL` i `FOUNDER_CONTRACT_NAME` są częścią kontraktu środowiska.

### 6.3 Plan „constitutional continuity" (skomitowane w `docs`)

Commit `88702f5 docs: plan constitutional continuity rollout`: nowy `docs/plans/resolution-continuity-implementation.md` + aktualizacja sprintu `.planfile/sprints/current.yaml` (±130 linii). Kierunek: ciągłość decyzyjna organizacji (digital twin) — spójny z nowym scope z 6.1.

### 6.4 Wpływ na rekomendowaną kolejność

Bez zmian dla punktu 4 (dedup ticketów / SELFDEV TTL — nadal następny krok ścieżki Plesk). Punkty 6.1 i 6.2 są już skomitowane, przetestowane w pełnej bramce platformy i uruchomione w lokalnym trybie live/mock.
