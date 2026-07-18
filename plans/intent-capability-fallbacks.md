# Design: intent packs + recipe policy (fallbacks without orchestration bugs)

Status: design note (no large code change). Complements `docs-subactor-com-publish.md`
and `autonomy-cli-runbook.md`.

## Verdict

Ease-of-intent is blocked less by OpenRouter and more by **N-way manual wiring**
plus a **linear fail-fast** recipe runner. Transport choice SFTP→FTP already lives
in the Plesk connector (`transport=auto`). What is missing is a single place to
declare **named intents** and **cross-URI ensure / optional / verify** policy so
authors do not re-route phrases, AQL, step-catalog, and recipes by hand.

## What is hard today (from code)

| Layer | Where | Pain |
| --- | --- | --- |
| Phrases | `agents/nlp-uri-phrases.yaml` **and** inline `PHRASES` in `docs-sync-intent.mjs` / `www-sync-intent.mjs` | Drift; comments admit “kept inline” |
| LLM catalog | `platform/config/llm-organization-intents.json` | Separate field schema from AQL |
| AQL model | `contracts/models/docs-httpdocs-sync.pl.aql` | Points at one `MODUŁY` name only |
| Step catalog | `platform/config/step-catalog.json` → `create_docs_httpdocs_sync_ticket` | Duplicates `uri_processes` from recipe |
| Recipe | `docs/deployment/*.urirun.json` | Linear list; no `on_fail` / `optional` |
| Planfile import | `.planfile/imports/*` + deployment YAML | Fourth copy of the same steps |
| Bot allow | `ALLOW MODEL` / `ALLOW URI_PROCESS` in contract AQL | Easy to forget when adding a model |
| Orchestrator | `runTask` in `@subactor/orchestrator` | Topo-order then **halt on first urirun failure** |
| Transport fallback | `urirun-connector-plesk` `_TRANSPORT_ORDER` | Correct place — do **not** duplicate as two sync steps |

Misroute example: without deterministic phrase hit, control LLM path historically
defaults toward onboarding models — hence `resolveDocsSyncIntent` before OpenRouter.

## Recommended model (three layers)

```text
Intent pack (SSOT for “what the user meant”)
    → expands → phrases + AQL stub + llm fields + step-catalog ref + ALLOW MODEL
Recipe policy graph (SSOT for “how steps relate”)
    → expands → flat uri_processes for today’s runner (or native policy interpreter)
Connector capability (SSOT for “which transport/tool inside one URI”)
    → e.g. transport=auto (SFTP then FTP) — never invented by LLM
```

### 1. Intent pack (declare once)

New file shape, e.g. `platform/config/intent-packs/docs-httpdocs-publish.json`
(or YAML). One pack = one founder goal.

```json
{
  "id": "docs-httpdocs-publish",
  "aql_model": "docs-httpdocs-sync.pl.aql",
  "phrases": [
    "opublikuj docs na docs.subactor.com",
    "sync docs to docs.subactor.com"
  ],
  "situation_defaults": {
    "source_dir": "/home/tom/github/subactor/docs",
    "host": "prototypowanie.pl",
    "domain": "docs.subactor.com"
  },
  "recipe": "docs/deployment/docs-httpdocs-sync.urirun.json",
  "step_module": "create_docs_httpdocs_sync_ticket",
  "allow_uri_processes": ["plesk://*"]
}
```

Generators (or a small loader) feed phrase resolvers + llm-organization-intents +
AQL `MODUŁY` wiring. Control plane imports **one** pack list instead of parallel
`PHRASES` arrays.

### 2. Recipe policy (orchestrator owns interpretation)

Extend `UriProcess` (today: `id, uri, payload, depends_on, human_approval`) with
optional policy fields. Keep expansion compatible with Planfile tickets.

```json
{
  "id": "docs-httpdocs-publish",
  "situation": { "domain": "docs.subactor.com", "source_dir": "…/docs" },
  "uri_processes": [
    {
      "id": "ensure-domain",
      "uri": "plesk://host/api/query/request",
      "payload": { "path": "/api/v2/domains" },
      "on_fail": "continue",
      "optional": true
    },
    {
      "id": "ensure-ftpuser",
      "uri": "plesk://host/ftpuser/command/ensure",
      "payload": {
        "domain": "docs.subactor.com",
        "vault_entry_id": "plesk-sftp"
      },
      "depends_on": ["ensure-domain"],
      "on_fail": "halt",
      "human_approval": true
    },
    {
      "id": "probe-transports",
      "uri": "plesk://host/site/query/methods",
      "depends_on": ["ensure-ftpuser"]
    },
    {
      "id": "sync-dry-run",
      "uri": "plesk://host/site/command/sync",
      "payload": {
        "transport": "auto",
        "apply": false,
        "domain": "docs.subactor.com",
        "source_dir": "/home/tom/github/subactor/docs",
        "remote_path": "/httpdocs"
      },
      "depends_on": ["probe-transports"]
    },
    {
      "id": "sync-apply",
      "uri": "plesk://host/site/command/sync",
      "payload": { "transport": "auto", "apply": true },
      "depends_on": ["sync-dry-run"]
    },
    {
      "id": "verify-https",
      "uri": "httpscheck://docs.subactor.com/",
      "depends_on": ["sync-apply"],
      "optional": true,
      "on_fail": "ticket"
    }
  ]
}
```

Semantics for `runTask`:

| Field | Meaning |
| --- | --- |
| `on_fail: halt` | Current behavior (default) |
| `on_fail: continue` | Record failure, run dependents that do not require success |
| `on_fail: ticket` | Open/escalate Planfile ticket; stop live chain |
| `optional: true` | Failure does not fail the whole plan |
| `transport: auto` | Connector-local SFTP→FTP — **not** two recipe sync steps |

Optional later: `strategy: try_in_order` group for **cross-URI** alternatives
(e.g. two different ensure URIs). Do not use this for SFTP vs FTP.

### 3. OpenRouter role

| OpenRouter may | OpenRouter must not |
| --- | --- |
| Pick **named** intent pack / AQL model id | Invent URI DAGs or fallback chains |
| Fill situation fields (`source_dir`, `domain`, …) | Choose SFTP vs FTP or vault ids ad hoc |
| Decompose multi-goal NL into **ordered pack ids** | Call connectors or see secrets |

Deterministic phrase map stays first (avoids onboarding misroute). LLM is
fallback for unknown wording → pack id + fields. Fallbacks and ensure order are
**policy in the recipe**, executed by orchestrator + connectors.

### Where each concern lives

| Concern | Home | Not here |
| --- | --- | --- |
| Named intent + phrases + defaults | Intent pack → feeds AQL / llm catalog / phrases | Orchestrator |
| Who may run which URI/model | Contract AQL `ALLOW *` | Recipe prose |
| Step order, optional, on_fail | Recipe `uri_processes` + orchestrator interpreter | OQL store as SSOT |
| OQL | Thin `process.run` record per step (already) | Policy graph |
| Transport SFTP→FTP | Connector `transport=auto` | Recipe duplicate steps |
| Apply safety | `PLESK_SYNC_APPLY` + dry-run default | LLM |

## Minimal implementation checklist

1. **Intent pack schema + one pack** for `docs-httpdocs-publish`; load phrases from pack in control (`docs-sync-intent`) and stop dual-maintaining YAML/inline lists (generate or share one module).
2. **Extend `UriProcess`** with `optional` / `on_fail`; teach `runTask` continue/ticket semantics (regression test: ensure optional fail → sync still runs).
3. **Grow docs recipe** toward runbook target: ensure-ftpuser → methods → dry-run → apply → optional https-check (keep `transport: auto`).
4. **Timeouts**: align orchestrator/urirun exec timeout with FTP upload (live apply failed ~30s in publish plan) — ops, not LLM.
5. **Do not** move transport fallback into recipes; **do not** let OpenRouter emit arbitrary `uri_processes`.

## Relation to docs.subactor.com publish

Current live recipe is methods → dry-run → apply. Transport fallback already
works when FTP is available; apply failed on subprocess timeout. Intent routing
works when phrase hits `resolveDocsSyncIntent`. Next product step for “easy
intents” is pack SSOT + recipe `on_fail`/`optional`, not a smarter LLM planner.
