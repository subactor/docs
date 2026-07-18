# ADR-005: Rollback i failure semantics

- **Status:** Proposed  
- **Data:** 2026-07-18  
- **Kontekst:** [`../autonomy-recommended-solution.md`](../autonomy-recommended-solution.md) §5, §8, §11  
- **Pytanie statusowe:** „Rollback / failure semantics?”

## Decyzja

1. **Deploy release-based** — nie destrukcyjny sync do aktywnego `/httpdocs`; aktywacja atomowa (`current` / `previous`).
2. **Recipe policy** (`on_fail`): `halt` (domyślne, legacy) | `continue` | `ticket` | `rollback`.
3. Rollback techniczny: `activate(previous_release)` → verify → `rolled_back` + ticket.
4. Rollback DNS (boundary): przy nieudanym cutoverze przywrócić poprzedni desired state (np. Pages).
5. Stany planu: pełny lifecycle (`proposed` … `completed`) oraz stany błędów (`applied_unverified`, `rolled_back`, `needs_human`, …) — bez redukcji do `ok: true/false`.

## Konsekwencje

- Connector jest właścicielem transportu i rollbacku technicznego.
- Orchestrator jest właścicielem semantyki `on_fail` / zależności (`depends_on` vs `after`).
- Częściowy upload / hash mismatch blokuje aktywację.
