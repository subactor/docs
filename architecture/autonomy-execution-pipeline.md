---
{
  "schema": "subactor.doc/v1",
  "id": "docs.architecture.autonomy-execution-pipeline",
  "version": 1,
  "status": "current",
  "updated": "2026-07-30"
}
---

# Tor wykonania autonomii: NL → DSL → Task → URI → Twin → dry-run → apply → EQL / rollback

Kanoniczny model **docelowy i obowiązujący** dla mutacji w Subactor. Nie zastępuje
pomiaru stanu w [autonomy-loop-and-twin.md](./autonomy-loop-and-twin.md); ten
dokument mówi, **jak ma wyglądać poprawny łańcuch**, a tamten — **gdzie live
jeszcze się rwie**.

## 1. Jedno zdanie

Człowiek mówi po polsku/angielsku; LLM w trybie `require-llm` proponuje
**wersjonowany DSL / intent pack** (nie URI); Control materializuje **Task** z
exact URI z katalogu zdolności; **uri-twin** tylko **obserwuje** i wiąże fakty;
connector robi **dry-run → `plan_hash` → grant → apply**; **EQL** i zewnętrzny
read-back potwierdzają skutek; porażka mutacji → **retry / research / escalate /
rollback** (rollback failed = Founder, bezwarunkowo).

## 2. Etapy (kolejność twarda)

```text
NL (Founder / operator)
  │  require-llm, structured output, status=proposed
  ▼
DSL / intent pack (+ situation slots)
  │  walidacja schematu, canonical hash; człowiek lub policy grant akceptuje
  ▼
Task / Planfile ticket
  │  AQL scope, queue, actor, EQL, OQL; bez sekretów w treści
  ▼
URI Process (exact z katalogu / strategy binding)
  │  LLM NIE inventuje URI, vault id, transportu (ADR-001)
  ▼
Twin observe (uri-twin + plesk://…/query/…)
  │  docroot, DNS authority, subscription snapshot, bindings surface
  │  Twin = sytuacja i preconditions — NIE authority mutate (ADR-012)
  ▼
Payload validate vs inputSchema (research / capability-surface)
  │  zanim jakikolwiek HTTP do connectora — łapie domain vs site_id / zone
  ▼
dry-run (apply=false) → plan_hash (+ input_hash)
  │
  ▼
APPROVE(plan_hash) — single-use grant; master gates nadal obowiązują
  │
  ▼
apply (apply=true + grant + env gate np. PLESK_*_APPLY)
  │
  ▼
EQL + niezależny read-back (public DNS, HTTPS, content hash)
  │
  └─ on failure: RETRY (gdy input/plan się zmienił) | RESEARCH | ESCALATE | ROLLBACK
```

## 3. Mapowanie na warstwy

| Etap | Warstwa | Artefakt / kod |
| --- | --- | --- |
| NL→DSL | Founder chat / `require-llm` | Control conversation + LLM Gateway |
| Intent | Intent packs | `platform/config/intent-packs/` |
| Strategy | Strategy DSL | `problem-strategies/catalog.v1.json`, `strategy-dsl.mjs` |
| Task | Planfile | process ticket + AQL/OQL/EQL |
| URI | Connector | `plesk://…`, `httpcheck://…` |
| Twin | uri-twin + Control normalize | `plesk-twin-sources.mjs`, query routes |
| Payload SSOT | Control helpers | `plesk-uri-payloads.mjs` |
| dry-run/apply | urirun-connector-plesk | command routes + apply grant |
| EQL | runtime / tickets | outcome receipts, public probes |

## 4. Niezmienniki

1. **LLM nie tworzy URI, vault id, transportu ani polityki apply** (ADR-001).
2. **Twin nie mutuje** — `query/*` i twin-fact nie dają `apply` (ADR-012).
3. **DNS management plane** przy `cloudflaredns`: Plesk jest SSOT planu rekordów;
   token Cloudflare API **nie** jest domyślną ścieżką
   (`knowledge://subactor/ops.plesk.cloudflare-dns-management/v2`).
4. **Payloady** muszą używać nazw ze snapshotu bindings (`zone`, `site_id`,
   `subscription`, `subdomain`+`parent_domain`) — nie aliasów `hostname`/`domain`
   tam, gdzie connector ich nie przyjmuje.
5. **Mutacja** zawsze: dry-run → exact `plan_hash` → grant → env gate → apply → EQL.
6. **HITL** tylko na granicach POLICY (sekret, grant, konflikt, wyczerpany budżet),
   nie na każdym observe.
7. **Cloudflare Tunnel** nie jest domyślnym deployem originu Plesk; public
   reverse-proxy wymaga publicznego HTTPS upstreamu z auth (nie loopback).

## 5. Przykład Plesk-native (founder.subactor.com)

```text
Observe:
  plesk://host/doctor/query/report
  plesk://host/subscription/query/snapshot   {subscription:"subactor.com"}
  plesk://host/dns/query/authority           {zone:"subactor.com"}
  plesk://host/site/query/docroot            {domain:"founder.subactor.com"}
  plesk://host/dns/query/records             {site_id:185}

Dry-run mutate:
  plesk://host/site/command/subdomain-ensure
    {parent_domain:"subactor.com", subdomain:"founder", www_root:"founder.subactor.com", apply:false}
  plesk://host/dns/command/replace
    {site_id:185, host:"founder.subactor.com", record_type:"A", value:"217.160.250.222", apply:false}
  plesk://host/site/command/sync
    {domain, source_dir, remote_path, apply:false}  → plan_hash

Apply (po granicie + PLESK_SYNC_APPLY / DNS gate):
  ten sam plan_hash, apply:true

EQL:
  public A consensus, HTTPS SAN, content hash
```

## 6. Co jest kodem, a co polityką

| | |
| --- | --- |
| Kod deterministyczny | schematy, hashe, binding URI, payload builders, gates, receipts |
| LLM | wybór packa / slotów z NL; status zawsze `proposed` do akceptacji |
| Founder | sekrety, grant `plan_hash`, konflikty POLICY, failed rollback |
| Twin | świeżość faktu, refuse → `precondition-blocked` przed command |

## 7. Powiązane

- [ADR-001](./adr/001-autonomy-scope.md) — scope / intent packs
- [ADR-012](./adr/012-uri-twin-observe-layer.md) — twin observe vs mutate
- [autonomy-loop-and-twin.md](./autonomy-loop-and-twin.md) — live pętla i research
- [autonomy-definition](../../platform/config/knowledge/entries/architecture.autonomy-definition.v1.md)
- Plan refaktoru: [../plans/autonomy-pipeline-refactor-2026-07-30.md](../plans/autonomy-pipeline-refactor-2026-07-30.md)
