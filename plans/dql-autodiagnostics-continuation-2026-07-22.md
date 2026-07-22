---
{
  "schema": "subactor.doc/v1",
  "id": "docs.plans.dql-autodiagnostics-continuation-2026-07-22",
  "version": 1,
  "status": "current",
  "updated": "2026-07-22"
}
---

# Plan kontynuacji — DQL i pętla Doctor/Repair/Validator

## Stan wykonany

- schema `subactor.diagnostic-profile/v1` i kanoniczny fixture;
- deterministyczny evaluator `subactor.diagnostic-report/v1`;
- operatory `expression`, `all`, `dependency_graph`, `immutable_refs`;
- stabilne fingerprinty i governed Problem Profile candidates;
- aktywny profil architektury Control oraz read-only collector;
- zielony live-static read-back 3/3 i bramka `test:meta`.

## P1 — Diagnostic Registry

Załadować wyłącznie aktywne, zwalidowane profile z Artifact Registry. Każdy
profil musi mieć immutable revision URI, ownera, AQL obserwatora i maksymalny
budżet snapshotu. Zmiana treści wymaga nowej wersji.

## P2 — typowane adaptery snapshotu

Zastąpić skan literalnego kodu runtime route registry. Dodać adaptery dla
Planfile lifecycle, connector doctor, Artifact Registry, SODL, Process Packów,
DNS/site resources i desired-state. Adaptery są read-only i nie zwracają
sekretów.

## P3 — czas i przyczynowość

Dodać `fresh_within`, `eventually_within`, `always_during` i bounded event
windows nad SODL. Fakty, korelacje i hipotezy przyczyn muszą być oddzielnymi
polami; DQL nie może deklarować hipotezy jako potwierdzonej przyczyny.

## P4 — Finding Outbox i ProblemCase

Zapisywać raport przez idempotentny outbox. Deduplikować occurrence po
fingerprint i snapshot hash, tworzyć ProblemCase dopiero według polityki oraz
zachować niezależne RepairCandidate dla alternatywnych napraw.

## P5 — Doctor/Repair/Validator

Doctor tworzy snapshot i finding. Repair działa w izolowanym worktree wyłącznie
z ticketem, AQL i grantem. Niezależny Validator uruchamia EQL na świeżym
snapshotcie. Tylko receipt może zamknąć ProblemCase.

## P6 — scheduler i obserwowalność

Dodać lease, budżety częstotliwości, backoff, dashboard statusów profili oraz
kanały odpowiedzialnych aktorów. Awaria collectora pozostaje widocznym
problemem diagnostycznym i nie może być interpretowana jako stan zdrowy.

## Warunek pełnej autonomii diagnostycznej

Kontrolowany defekt musi przejść ścieżkę:

```text
snapshot → DQL finding → occurrence → ticket → RepairCandidate
         → URI Process → świeży snapshot → EQL receipt → verified
```

Bez receipt system może powiedzieć „wykryto” albo „podjęto próbę”, ale nie
„naprawiono”.
