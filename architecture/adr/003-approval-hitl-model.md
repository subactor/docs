# ADR-003: Model zatwierdzania i Human-in-the-loop

- **Status:** Proposed  
- **Data:** 2026-07-18  
- **Kontekst:** [`../autonomy-recommended-solution.md`](../autonomy-recommended-solution.md) §2.3, §6  
- **Pytanie statusowe:** „Kto zatwierdza apply?” / „HITL when?” / „Dry-run zawsze przed apply?”

## Decyzja

**Nie** zatwierdzać ręcznie każdego deployu. Podział klas operacji:

| Klasa | Przykład | Tryb |
| --- | --- | --- |
| Read-only | DNS query, methods, health | automatyczny |
| Reversible mutate | publikacja kolejnego release | automatyczny **po obowiązkowym dry-run** |
| Boundary change | nowa domena, DNS, utworzenie credential | **jednorazowy HITL** |
| Governance change | nowy intent pack, AQL allow, polityka | **obowiązkowy review człowieka** |

Po bootstrapie domeny/credentials/polityki publikacja docs = **zero-touch**.

### Bramki apply (warstwa techniczna)

1. Master kill switch: `AUTONOMY_MUTATIONS_ENABLED=1` (oraz istniejący `PLESK_SYNC_APPLY` jako kill switch domenowy).
2. Podpisany, krótko żyjący **apply grant** związany z `run_id`, `plan_hash`, artefaktem, targetem, aktorem.
3. Dry-run tworzy **immutable plan/manifest** — apply nie przelicza listy plików.

## Konsekwencje

- Founder bypass (`SUBACTOR_ADMIN_TOKEN`) nie zastępuje grantu w docelowym modelu produkcyjnym.
- Bot bez grantu i bez kill switcha nigdy nie mutuje.
