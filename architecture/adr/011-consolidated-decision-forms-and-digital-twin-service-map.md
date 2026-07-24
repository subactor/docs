---
{
  "schema": "subactor.doc/v1",
  "id": "docs.architecture.adr.011-consolidated-decision-forms-and-digital-twin-service-map",
  "version": 1,
  "status": "current",
  "updated": "2026-07-24"
}
---

# ADR 011: Skonsolidowane Formularze Decyzyjne oraz Mapa Usług Digital Twin

## Status

Zaakceptowany (Accepted)

## Kontekst

W miarę rozwoju systemu autonomii Subactor, w panelu sterowania Foundera (Control) pojawiły się interaktywne formularze decyzji dotyczące ticketów (np. `PLF-1281`). W poprzedniej wersji interfejs prezentował dwa niezależne pola tekstowe:
- `cancellation_reason` („Powód anulowania”)
- `decision_context` („Dodatkowe wyjaśnienie decyzji”)

Prowadziło to do zbędnej redukcji czytelności UX oraz powielania wpisów w audit logu.

Równolegle rozwinięto architekturę **Digital Twin** dla usług publicznych (`founder.subactor.com`, `autonomicznosc.pl`, `status.subactor.com`, `www.subactor.com`, `docs.subactor.com`, `contracts.subactor.com`). Część ticketów automatycznych błędnie zakładała wymóg `managed_tunnel_credential_required` dla serwisów, dla których Founder wyznaczył tryb `public_ingress_mode=plesk_public_origin` z `tunnel_mode=none`.

## Decyzja

1. **Konsolidacja Formularzy Decyzyjnych (`decision_context`)**:
   - Wprowadzono jedno uniwersalne pole tekstowe `decision_context` („Notatka (opcjonalnie)”) do uzasadnienia dowolnego wyboru opcji radio (np. „Kontynuuj”, „Odłóż”, „Anuluj ticket”).
   - Dodano automatyczną dedupikację w `normalizeInteractiveForm` ([interactive-form.mjs](file:///home/tom/github/subactor/core/services/control/src/interactive-form.mjs)), która usuwa legacy pole `cancellation_reason`, gwarantując obecność dokładnie jednego pola notatki.
   - Zachowano pełną wsteczną kompatybilność w trasie backendowej (`applyFounderFormEffects`), przyjmującą opcjonalnie dawne odpowiedzi `cancellation_reason`.

2. **Mapa Usług Digital Twin & Brak Tunelu dla Plesk Ingress**:
   - Wprowadzono sformalizowany profil DOQL `public-site-service-map.doql.json` oraz `public-site-capability-inventory.doql.json`.
   - Zaktualizowano wszystkie manifesty projektów (`projekty/*/project.manifest.json`), jawnie deklarując `public_ingress_mode: "plesk_public_origin"` oraz `tunnel_mode: "none"`.
   - Stworzono skrypt uruchomieniowy `platform/scripts/run-public-site-service-map.mjs` do weryfikacji sytuacji bez mutacji środowiska.

3. **Gwarancje i Rejestr Artefaktów**:
   - Utrzymano wymóg 100% zwalidowanego rejestru artefaktów (`npm run artifacts:build && npm run artifacts:check`).
   - Wszystkie odwołania do bazy wiedzy wersjonowane są w sposób append-only (`knowledge://subactor/architecture.digital-twin.public-site-service-map/v3`).

## Konsekwencje

- Ujednolicony i czysty interfejs modalny w panelu Control Foundera.
- Brak fałszywych ticketów blokujących dla Cloudflare Tunnel tam, gdzie ruchem zarządza Plesk.
- Przejrzysty i powtarzalny proces weryfikacji readiness tras publicznych przy pełnym pokryciu testowym (`npm test` 730/730 PASS).
