---
{
  "schema": "subactor.doc/v1",
  "id": "docs.operations.ticket-trigger-recovery-guide",
  "version": 5,
  "status": "current",
  "updated": "2026-08-04"
}
---

# Ticket Trigger Event — przewodnik operacyjny

## Cel

`inputs.trigger_events` opisuje stan, na który czeka ticket. Kontroler sprawdza
go cyklicznie w sekret-free rejestrze Digital Twin i może automatycznie zamknąć
ticket albo ponownie skierować go do wykonania.

Nowy incydent może również zostać utworzony przez wersjonowaną politykę stanu.
W takim przypadku ticket otrzymuje `inputs.existence_contract`, który zapisuje
dokładny powód istnienia, politykę i epizod awarii. `purpose_context` nadal jest
wymagany i odpowiada za cel strategiczny; te pola nie zastępują się wzajemnie.

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

## Polityki tworzenia incydentów

Przenośne reguły są zapisane w
`platform/config/digital-twin/ticket-state-policies.json`, bez sekretów i bez
zależności od lokalnego `.env`. Warunki odwołują się do dokładnych zmiennych,
na przykład:

```text
state://subactor/soa/connector/plesk/configured
state://subactor/soa/connector/plesk/enabled
state://subactor/soa/connector/plesk/authenticated
state://subactor/soa/connector/plesk/state
```

`configured`, `enabled`, `reachable` i `authenticated` są odrębnymi faktami.
Stan `unknown` albo obserwacja starsza od limitu nie może utworzyć ani zakończyć
ticketu.

W stanie kontrolera sprawdź `ticket_trigger_cron.state_policies`. Pełna,
uwierzytelniona projekcja jest dostępna pod
`GET /api/digital-twin/system-state` dla zakresu `plans:read`.

Reconciler tworzy najwyżej jeden ticket na epizod. Jeżeli aktywny starszy ticket
ma trigger dla tego samego exact SOA/POA, zostaje oznaczony jako pokrywający
problem i nowy duplikat nie powstaje. Automatyczne recovery kończy tylko ticket
związany z tą samą polityką; nie zamyka podobnie nazwanego zadania projektowego.

Producent ticketu deklaruje zależność podczas tworzenia. Ops Observer dodaje
trigger recovery dla 401/403, awarii health check i błędów dostępności Pleska,
ale nie dla 404 ani błędu konkretnej operacji biznesowej — aktywny connector nie
dowodzi wtedy usunięcia pierwotnego problemu.

## Governance jako stan

Digital Twin obserwuje również dwa wewnętrzne obiekty SOA:

```text
soa://subactor/integration/organization-intent-foundation
soa://subactor/integration/ticket-purpose-integrity
```

Pierwszy tworzy jeden ticket Foundera, gdy aktywna Konstytucja nie ma jeszcze
ratyfikowanego fundamentu wizja → misja → strategia. Drugi pozostaje
`not_configured`, dopóki fundament nie istnieje. Dopiero później publikuje
`gap_count`, `unbound_count` oraz `partial_count` i może utworzyć osobny ticket
uzgodnienia strategicznego celu aktywnej pracy.

Oba tickety pozostają ludzką granicą. System może przygotować preview, diff i
prompt pomocniczy, ale nie ratyfikuje wizji, misji ani strategii za Foundera.
Nieaktywne lub opcjonalne connectory nie dostają polityki tylko dlatego, że ich
stan wynosi `unknown` albo `not_configured`.

## Źródła

- `artifact://subactor/contracts/schemas/ticket-trigger-event.schema.v1.json`
- `artifact://subactor/contracts/schemas/digital-twin-state-registry.schema.v1.json`
- `artifact://subactor/contracts/schemas/system-state-variable-registry.schema.v1.json`
- `artifact://subactor/contracts/schemas/ticket-existence-contract.schema.v1.json`
- `artifact://subactor/contracts/schemas/ticket-state-policy-registry.schema.v1.json`
- `artifact://subactor/platform/config/digital-twin/ticket-state-policies.json`
- `knowledge://subactor/architecture.ticket-trigger-recovery/v3`
- `knowledge://subactor/architecture.ops-observer-state-dependent-incidents/v1`
- `knowledge://subactor/architecture.governance-digital-twin-ticket-purpose/v2`
