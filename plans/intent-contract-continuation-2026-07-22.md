---
{
  "schema": "subactor.doc/v1",
  "id": "docs.plans.intent-contract-continuation-2026-07-22",
  "version": 3,
  "status": "current",
  "updated": "2026-07-22"
}
---

# Plan kontynuacji — Intent Contract, lifecycle i równoległe naprawy

## Cel

Utworzyć jedno, wersjonowane źródło prawdy dla intencji zapisanej przez
człowieka lub maszynę, bez zastępowania istniejących AQL, OQL, EQL, URI Process,
Process Packów, SODL i Planfile.

## Stan wejściowy

### Postęp 2026-07-22

- P0 wykonane: zaakceptowano ADR-009 i dodano JSON Schema
  `subactor.intent-contract/v1`.
- P1 częściowo wykonane: Runtime ma deterministyczną kanonikalizację, SHA-256,
  walidację lifecycle, fail-closed secret/execution-field scan, projekcję
  Markdown/form oraz semantic diff z sygnałem ponownej akceptacji.
- Artifact Registry v2 obejmuje teraz również kanoniczne schemy z repozytorium
  Contracts.
- Nadal otwarte w P1: zapis nowej wersji z formularza, registry/API instancji
  oraz test zgodności hasha między implementacjami.

### Wykonane

- Process Pack i `subactor.process-envelope.v2` wiążą AQL, EQL, OQL i kroki URI.
- Runtime ma współdzielony, fail-closed kontrakt readiness ticketu.
- Control oferuje obserwacyjną reconciliację oraz egzekwowane przejście
  pojedynczego ticketu do `ready`.
- Completion receipt zachowuje `expected`, verifier oraz `verified_by` kroków.
- SODL/1 zapisuje odwracalne zdarzenia Planfile i Control z korelacją.
- Strategy DSL wybiera zatwierdzone strategie i capabilities, bez generowania
  URI przez LLM.
- Problem Reaction ma Problem Profile, fingerprint, deduplikację obserwacji i
  klasyfikację reakcji bez automatycznej mutacji.
- Planfile posiada atomowy, idempotentny zapis evidence dla efektów zewnętrznych;
  Bridge rozróżnia skuteczny efekt od opóźnionego zapisu dowodu.

### Otwarte luki

- brak instancji intencji zachowującej wypowiedzi człowieka i maszyny oraz ich
  zaakceptowaną normalizację;
- bezpośrednie UI/API Planfile może zapisać `ready` bez wspólnego preflightu;
- Problem Reaction grupuje obserwacje po fingerprint, ale nie ma `ProblemCase`,
  `Occurrence` i konkurencyjnych `RepairCandidate`;
- pętla `reaction → ticket → URIrun → EQL → completion` pozostaje obserwacyjna;
- brak formularza pokazującego semantic diff przed akceptacją intencji.

## P0 — ADR i schema Intent Contract v1

Status: **wykonane**.

Rezultat:

- ADR potwierdzający, że Intent Contract jest kopertą danych, nie nowym runtime
  DSL;
- JSON Schema `subactor.intent-contract/v1`;
- statusy, reguły wersjonowania, canonical JSON i `contract_hash`;
- typy statementów człowieka, bota i systemu;
- jawny zakaz sekretów, caller-controlled URI oraz samonadawania authority.

Kryterium zakończenia: pozytywne i negatywne fixtures przechodzą meta-schema,
canonicalization i secret scan.

## P1 — parser, renderer i registry

Status: **w toku** — ukończono walidator, kanonikalizację i hash Runtime.

Rezultat:

- biblioteka Runtime walidująca i haszująca kontrakt;
- deterministyczny renderer JSON → Markdown/form oraz round-trip formularza do
  nowej wersji JSON;
- wpisy Artifact Registry z immutable revision URI;
- API Control do preview, diff i pobrania wersji bez mutacji wykonawczej.

Kryterium zakończenia: Node i ewentualny Python generują identyczny hash, a
renderer nie zmienia danych kanonicznych.

## P2 — integracja z Process Pack i Planfile

Rezultat:

- zaakceptowany Intent Contract wybiera istniejący Process Pack i typowane
  bindingi;
- ticket v2 przechowuje `intent_ref` oraz `intent_hash`;
- zmiana kontraktu unieważnia niezgodny `plan_hash` i apply grant;
- Planfile otrzymuje wersjonowany readiness policy hook wywołujący ten sam
  kontrakt Runtime dla UI, API i Control.

Kryterium zakończenia: nie da się utworzyć false-ready ani wykonać planu
związanego ze starszą wersją intencji.

## P3 — współautorstwo człowieka i maszyny

Rezultat:

- formularz pokazuje oryginalne statements, proponowaną normalizację,
  niejednoznaczności, constraints i acceptance criteria;
- poprawka człowieka tworzy nowy statement oraz semantic diff;
- akceptacja jest związana z principalem, timestampem i hashem;
- maszyna może ponownie zinterpretować tekst, ale nie nadpisuje zaakceptowanej
  wersji.

Kryterium zakończenia: audyt pozwala jednoznacznie odpowiedzieć kto co napisał,
co zaproponował model, co poprawił człowiek i która wersja została wykonana.

## P4 — ProblemCase i równoległe RepairCandidate

Rezultat:

- schemy `ProblemCase`, `Occurrence`, `RepairCandidate` i relacje
  `derived_from`, `alternative_to`, `conflicts_with`, `supersedes`,
  `resolved_by`;
- osobny fingerprint obserwacji, `problem_id` przyczyny i `candidate_id`
  sposobu naprawy;
- izolowane branche/worktree oraz jeden lease promocji na mutowany zasób;
- wspólny EQL dla kandydatów i niezależne receipts walidatora.

Kryterium zakończenia: dwie różne naprawy mogą być testowane równolegle bez
scalenia ich jako duplikatów i bez równoległej mutacji produkcji.

## P5 — doctor → repair → validator

Rezultat:

- Doctor zapisuje occurrence, dowody i hipotezę, ale nie deklaruje sam przyczyny
  jako faktu;
- Repair działa wyłącznie z autoryzowanego ticketu kandydata i ograniczonego
  worktree/capability;
- Validator uruchamia wspólne EQL na świeżym baseline i emituje receipt;
- promocja wybiera jednego zwycięzcę albo jawnie kompatybilny zestaw;
- kandydaci niewybrani otrzymują `rejected` lub `superseded`, zachowując wiedzę.

Kryterium zakończenia: kontrolowany test błędu przechodzi pełną pętlę do receipt,
a ponowienie nie tworzy drugiego efektu zewnętrznego.

## P6 — migracja i obserwacja

Rezultat:

- nowe zadania otrzymują Intent Contract obowiązkowo;
- legacy tickety są migrowane jako oznaczone projekcje, bez przepisywania
  historycznych wypowiedzi;
- dashboard pokazuje intencję, ticket, problem, kandydata, wykonanie i receipt;
- metryki obejmują clarification rate, stale intent, false-ready, superseded
  repairs oraz czas od occurrence do zweryfikowanego rozwiązania.

Kryterium zakończenia: read-back wykazuje pełne referencje i brak niejawnej
mutacji poza AQL/ticket/grant.

## Kolejność realizacji

P0 i P1 są warunkiem dalszej integracji. P2 zamyka lukę lifecycle u źródła.
P3 można rozwijać równolegle z P4 po zamrożeniu schematu v1. P5 wymaga P2 i P4.
P6 rozpoczyna się dopiero po pozytywnym pilocie P5.

Każda faza powinna otrzymać osobny ticket z Process Envelope v2, właścicielem,
EQL, exact URI lub jawną granicą pracy człowieka oraz completion receipt.
