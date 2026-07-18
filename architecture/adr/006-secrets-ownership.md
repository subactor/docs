# ADR-006: Ownership sekretów

- **Status:** Proposed  
- **Data:** 2026-07-18  
- **Kontekst:** [`../autonomy-recommended-solution.md`](../autonomy-recommended-solution.md) §9  
- **Pytanie statusowe:** „Ownership sekretów?” / „Ensure SFTP → vault?”

## Decyzja

| Rola | Odpowiedzialność |
| --- | --- |
| Człowiek | tworzy i rotuje credential (bootstrap / boundary HITL) |
| Vault | jedyny **SSOT** sekretu |
| Recipe | tylko logiczny `credential_ref` |
| Runtime | krótkotrwały lease; unieważnienie po runie |

Zachowanie:

```text
credential istnieje → lease test → kontynuuj automatycznie
credential nie istnieje → needs_human + ticket „bootstrap credential” → bez publikacji
```

System **nie** pyta LLM o hasło i **nie** zapisuje sekretów w ticketach, recipe ani logach NL.

## Konsekwencje

- Ensure SFTP/FTP → vault jest krokiem preflight (capability), nie częścią free-form LLM.
- Kryterium D9 statusu pozostaje twarde.
- Tokeny w lokalnym `.env` nie zastępują modelu vault w produkcji.
