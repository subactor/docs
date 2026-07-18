# Design note: intent packs + recipe policy

**Status:** skrót / pointer.  
**Dokument kanoniczny:**
[`../architecture/intent-orchestration-and-fallbacks.md`](../architecture/intent-orchestration-and-fallbacks.md)

Ten plik pozostaje jako krótka nota planowa. Pełna analiza stanu stacku,
gap analysis, model generyczny (intent packs, recipe policy, connector
capabilities, rola LLM), szkice schematów, migracja, non-goals i acceptance
criteria są w dokumencie architektury powyżej.

## Werdykt (1 akapit)

Ease-of-intent blokuje **N-krotne ręczne wiring** oraz **liniowy fail-fast**
w `runTask`, nie brak OpenRouter. Fallback transportu (np. SFTP→FTP przy
`transport=auto`) należy do konektora; `optional` / `on_fail` / `try_in_order`
— do recipe + orchestratora; named intent + situation — do intent pack SSOT.
LLM wybiera pack id i sloty, nie URI DAG.

## Uzupełnienia

- Runbook CLI: [`../autonomy-cli-runbook.md`](../autonomy-cli-runbook.md)
- Przykład publish docs: [`docs-subactor-com-publish.md`](docs-subactor-com-publish.md)
