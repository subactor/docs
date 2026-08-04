---
{
  "schema": "subactor.doc/v1",
  "id": "docs.operations.ticket-trigger-recovery-guide",
  "version": 1,
  "status": "current",
  "updated": "2026-08-04"
}
---

# Ticket Trigger Event — przewodnik operacyjny

## Cel

`inputs.trigger_events` opisuje stan, na który czeka ticket. Kontroler sprawdza
go cyklicznie w sekret-free rejestrze Digital Twin i może automatycznie zamknąć
ticket albo ponownie skierować go do wykonania.

## Przykład SOA: systemowy Plesk

```json
{
  "schema": "subactor.ticket-trigger-event/v1",
  "id": "plesk-auth-recovered:plesk.domain.list",
  "event": "dependency.recovered",
  "dependency": {
    "kind": "soa",
    "ref": "soa://subactor/connector/plesk",
    "required_state": "active",
    "max_age_seconds": 600
  },
  "action": "ticket.complete",
  "explanation": "Systemowy profil Plesk ponownie przeszedł uwierzytelniony test."
}
```

Użyj `ticket.complete` wyłącznie wtedy, gdy celem ticketu jest oczekiwanie lub
diagnostyka dostępności. Jeżeli po odzyskaniu usługi trzeba wykonać kolejne
kroki, ustaw `ticket.resume`.

## Przykład POA: wewnętrzny proces

```json
{
  "schema": "subactor.ticket-trigger-event/v1",
  "id": "public-route-observed",
  "event": "dependency.recovered",
  "dependency": {
    "kind": "poa",
    "ref": "poa://subactor/process/public-route-observed",
    "process_uri": "httpcheck://host/http/query/status",
    "required_state": "succeeded",
    "max_age_seconds": 600
  },
  "action": "ticket.resume",
  "explanation": "Świeży receipt potwierdził dostępność publicznej trasy."
}
```

## Weryfikacja

W stanie kontrolera sprawdź sekcję `ticket_trigger_cron`. Dla automatycznie
zamkniętego ticketu historia musi zawierać aktora
`bot:ticket-trigger-cron-bot`, zdarzenie `ticket_trigger.recovered` i completion
receipt z assertion `trigger-dependency-recovered`.

Nie uznawaj samego `enabled=true` za recovery. SOA musi mieć stan `active`, a
obserwacja nie może być starsza niż `max_age_seconds`. Różne profile Pleska są
różnymi referencjami i nie zastępują się wzajemnie.

## Źródła

- `artifact://subactor/contracts/schemas/ticket-trigger-event.schema.v1.json`
- `artifact://subactor/contracts/schemas/digital-twin-state-registry.schema.v1.json`
- `knowledge://subactor/architecture.ticket-trigger-recovery/v1`
