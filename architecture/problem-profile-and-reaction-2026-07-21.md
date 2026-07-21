# Problem Profile v1 i reakcje na błędy

Status: wdrożona podstawa kontraktu błędu; automatyczne strategie mutujące
pozostają fail-closed  
Data: 2026-07-21

## Decyzja

Subactor używa RFC 9457 `application/problem+json` jako publicznego kontraktu
błędu. RFC 9457 zastępuje RFC 7807. Problem Profile nie jest kolejnym językiem
wykonawczym: opisuje wystąpienie problemu, natomiast istniejące warstwy nadal
dzielą odpowiedzialności:

- SODL zapisuje fakt, korelację, fingerprint i historię;
- AQL określa authority do diagnozy, naprawy i odbioru powiadomienia;
- OQL określa operację naprawczą;
- URI Process wybiera jednoznaczny connector/runtime;
- EQL weryfikuje rezultat;
- TestQL sprawdza kontrakt i scenariusze awaryjne;
- Planfile przechowuje pracę, ownera i completion receipt.

## Kontrakt

`subactor.problem.v1` zawiera pola RFC 9457: `type`, `title`, `status`, `detail`
i `instance`. Rozszerzenia Subactora to między innymi `code`, `legacy_code`,
`category`, `severity`, `retryable`, `security`, `component`, `resource`,
`correlation_id`, `ticket_id` oraz deterministyczny `fingerprint`.

Pole `error` pozostaje tymczasowo aliasem bezpiecznego `legacy_code`, aby
istniejące klienty mogły przejść na `code` bez jednoczesnego breaking change.
Dowolne komunikaty wyjątków, stack trace, sekrety i query string nie są
publikowane jako `detail`.

Kod kanoniczny ma format `subactor.<domain>.<component>.<condition>`. HTTP status
opisuje transport i nie służy samodzielnie do wyboru reakcji. Severity używa
wartości zgodnych z telemetryką (`warn`, `error`, `fatal`), a incydent
bezpieczeństwa jest osobną klasyfikacją, nie poziomem severity.

## Zdarzenie

Każdy nieobsłużony błąd Control jest normalizowany i zapisany jako
`problem.detected`. Jego linia SODL ma `kind=problem`, `mode=observe`,
`status=failed` i `replayable=false`. Ponowienie dotyczy dopiero zatwierdzonego
process packa naprawczego, nigdy samego wyjątku.

Fingerprint jest hashem kanonicznego zestawu:

```text
code + component + environment + resource
```

Pozwala grupować powtórzenia bez tworzenia osobnego ticketu i e-maila dla
każdego timeoutu.

## Następny przyrost

Kolejny process pack powinien wykonywać wyłącznie obserwację, deduplikację i
klasyfikację `problem.detected`. Osobny deterministyczny katalog połączy
klasy problemów z istniejącymi strategiami: retry z idempotency key, circuit
breaker, secure credential intake, kompensacja, containment albo eskalacja do
właściwej authority. LLM może porządkować zatwierdzone strategie, ale nie może
wymyślać kodu, URI, odbiorcy, authority ani operacji.

Automatyczny apply pozostaje zabroniony bez wcześniejszego ticketu, aktualnego
AQL/EQL, dry-run, dokładnego plan hash i jednorazowego grantu.
