---
{
  "schema": "subactor.doc/v1",
  "id": "docs.architecture.adr.011-consolidated-decision-forms-and-digital-twin-service-map",
  "version": 3,
  "status": "current",
  "updated": "2026-07-29"
}
---

# ADR 011: Skonsolidowane Formularze Decyzyjne oraz Mapa Usług Digital Twin

## Status

Zaakceptowany (Accepted) — uzupełniony o runtime kontrolera i live evidence 2026-07-24 (v2 dokumentu)

## Kontekst

W miarę rozwoju systemu autonomii Subactor, w panelu sterowania Foundera (Control) pojawiły się interaktywne formularze decyzji dotyczące ticketów (np. `PLF-1281`). W poprzedniej wersji interfejs prezentował dwa niezależne pola tekstowe:
- `cancellation_reason` („Powód anulowania”)
- `decision_context` („Dodatkowe wyjaśnienie decyzji”)

Prowadziło to do zbędnej redukcji czytelności UX oraz powielania wpisów w audit logu.

Równolegle rozwinięto architekturę **Digital Twin** dla usług publicznych (`founder.subactor.com`, `autonomicznosc.pl`, `status.subactor.com`, `www.subactor.com`, `docs.subactor.com`, `contracts.subactor.com`). Część ticketów automatycznych błędnie zakładała wymóg `managed_tunnel_credential_required` dla serwisów, dla których Founder wyznaczył tryb `public_ingress_mode=plesk_public_origin` z `tunnel_mode=none`.

## Decyzja

1. **Konsolidacja Formularzy Decyzyjnych (`decision_context`)**:
   - Wprowadzono jedno uniwersalne pole tekstowe `decision_context` („Notatka (opcjonalnie)”) do uzasadnienia dowolnego wyboru opcji radio (np. „Kontynuuj”, „Odłóż”, „Anuluj ticket”).
   - Dodano automatyczną dedupikację w `normalizeInteractiveForm` ([interactive-form.mjs](../../../core/services/control/src/interactive-form.mjs)), która usuwa legacy pole `cancellation_reason`, gwarantując obecność dokładnie jednego pola notatki.
   - Zachowano pełną wsteczną kompatybilność w trasie backendowej (`applyFounderFormEffects`), przyjmującą opcjonalnie dawne odpowiedzi `cancellation_reason`.

2. **Mapa Usług Digital Twin & Brak Tunelu dla Plesk Ingress**:
   - Wprowadzono sformalizowany profil DOQL `public-site-service-map.doql.json` oraz `public-site-capability-inventory.doql.json`.
   - Zaktualizowano wszystkie manifesty projektów (`projekty/*/project.manifest.json`), jawnie deklarując `public_ingress_mode: "plesk_public_origin"` oraz `tunnel_mode: "none"`.
   - Stworzono skrypt uruchomieniowy `platform/scripts/run-public-site-service-map.mjs` do weryfikacji sytuacji bez mutacji środowiska.

3. **Gwarancje i Rejestr Artefaktów**:
   - Utrzymano wymóg 100% zwalidowanego rejestru artefaktów (`npm run artifacts:build && npm run artifacts:check`).
   - Wiedza append-only: aktualnie `knowledge://subactor/architecture.digital-twin.public-site-service-map/v4` (v3 superseded).

4. **Runtime kontrolera (doprecyzowanie 2026-07-24)**:
   - Etap `managed_tunnel_followup` supersede’uje tickety credential przy `tunnel_mode=none` (`supersession_reason=public_site_binding_tunnel_mode_none`).
   - Odblokowuje tickety źródłowe na `application_route_not_ready` (nie na tunnel).
   - Filtr credential jest wąski (label/nazwa intake) — nie wolno anulować ticketów deployment.
   - Remediacja plesk-origin nie czeka na managed-tunnel i nie powinna trzymać kroku `provide-upstream`.
   - `hr-control` montuje live `core/services/control/src` (RO), aby uniknąć stale image po poprawkach policy.

## Konsekwencje

- Ujednolicony i czysty interfejs modalny w panelu Control Foundera.
- Brak fałszywych ticketów blokujących dla Cloudflare Tunnel tam, gdzie ruchem zarządza Plesk.
- Przejrzysty i powtarzalny proces weryfikacji readiness tras publicznych (service-map API + unit tests control).
- **Nie** eliminuje błędów wykonania Pleska (proxy/upstream/treść) — te pozostają na trasie `application_route_not_ready` z dry-run i master gate.
- Live: PLF-1272 supersede OK; founder.subactor.com nadal default page Pleska (blocker treści, nie tunnel).

## Odwołania

- Ops: `docs/operations/digital-twin-service-map-operations-guide.md` (v2)
- Knowledge: `knowledge://subactor/architecture.digital-twin.public-site-service-map/v4`
