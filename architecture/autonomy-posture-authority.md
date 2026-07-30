---
{
  "schema": "subactor.doc/v1",
  "id": "docs.architecture.autonomy-posture-authority",
  "version": 1,
  "status": "current",
  "updated": "2026-07-30"
}
---

# Władza nad postawą autonomii: gdzie jest, kto ją ma, i dlaczego się rozjeżdża

Dokument opisuje stan zmierzony 30 lipca 2026. Wszystkie liczby i godziny pochodzą
z obserwacji działającej instancji.

## 1. Postawa zmieniła się dziś trzy razy, i za każdym razem po cichu

| godzina | nakładki compose kontenera | `mode` | PLF-2051 mówi |
|---|---|---|---|
| rano | — | consumers=1 | `execute_full` / `implemented` |
| 13:11 | + `control-safe.yml` | `observe_only` | `execute_full` / `implemented` |
| 16:43 | tylko `docker-compose.yml` | `execute_full` | `execute_full` / `implemented` |
| 17:20 | + `control-safe.yml` | `observe_only` | `execute_full` / `implemented` |

Decyzja Foundera nie zmieniła się ani razu. Zmieniło się to, z jakimi nakładkami
odtworzono kontener. **Ticket przez cały czas twierdzi, że decyzja jest wdrożona
i zweryfikowana.**

To nie jest incydent. Trzy przełączenia w ciągu jednego dnia roboczego, żadne
odnotowane, to wzorzec.

## 2. Dlaczego to było niewidoczne

Founder ma jedną drogę zapisu: formularz z jednorazowym tokenem, związany
z konkretnym ticketem (`POST /api/founder/form/submit`). Czat jest wyłącznie do
odczytu — `requireScope(req, "plans:read")`, dwa razy, i nic więcej; agent czatu
produkuje **linki**, nie mutacje.

Bramki, które tę decyzję realizują, ustawiają natomiast nakładki compose. Te dwie
rzeczy nie miały ze sobą żadnego połączenia:

```
decyzja  →  ticket Planfile   (founder_decisions, decision_status)
bramki   →  .env + nakładki   (AUTONOMOUS_QUEUE_*, AUTONOMY_MUTATIONS_*)
                ↑
         nic tego nie uzgadniało
```

Gorzej: `/health` **twierdził, że uzgadnia**. `controlSafetyStatus` wyliczał `mode`
ze zmiennych środowiskowych, po czym deklarował `authority_source: "founder_ticket"`
jako literał — nie otwierając żadnego ticketu. O 13:47 ręczył więc za władzę
Foundera dla postawy, którą nakładka nadpisała.

## 3. Co zostało zbudowane

### Warstwa: `subactor.control-safety-status/v3`

Pole `authority_source` istniało od v2 i było właściwym miejscem — musiało tylko
mówić prawdę. v3 deklaruje `environment`, bo to jest to, co funkcja czyta, i dokłada
dwa jawne sloty:

```json
"authority_source": "environment",
"authority_verified": false,
"decision_ref": null
```

Handler `/health` musi zostać tani i bez zależności, więc **deklaruje, co odczytał,
i nie ręczy za nic więcej**.

### Reconciler i sytuacja w DOQL

`autonomy-posture-reconciler.mjs` czyta decyzję z Planfile, porównuje z bramkami
i wypełnia te sloty. Wynik jest wyrażony jako **profil sytuacyjny DOQL**
(`autonomy.posture`) — ten sam format, którym opisane są `public-site.service-map`
i `organization-situation`, więc odpowiedź jest zapytywalna tam, gdzie wszystkie
pozostałe sytuacje.

```
GET /api/autonomy/posture   (projects:read, read-only)

assessments:  decision_authority | claim_integrity | external_exposure
kandydaci:    reconcile_posture_with_decision, correct_implementation_claim,
              record_decision_for_current_posture, name_unknown_decision_vocabulary
policy:       read_only: true, automatic_mutation_allowed: false
```

Odczyt o 17:24, minuty po wpięciu:

```json
{"authority_source":"environment","authority_verified":false,"decision_ref":"PLF-2051"}
{"decision_authority":"overridden_outside_decision",
 "claim_integrity":"ticket_claims_implemented_while_overridden"}
```

## 4. Trzy decyzje projektowe, które okazały się nieoczywiste

**`honoured` jest trójwartościowe, nie boolean.** „Nie dało się ustalić" to osobna
i wykonalna odpowiedź. Sprowadzenie do `false` zgłasza naruszenie, którego nikt nie
naprawi; sprowadzenie do `true` ręczy za coś niesprawdzonego — czyli powtarza błąd v2.

**Niezmapowana decyzja to luka, nie wartość domyślna.** Decyzja spoza tabeli
decyzja→bramki daje `unknown` i własnego kandydata (`name_unknown_decision_vocabulary`),
zamiast cicho przejść jako zgodna.

**„Autoryzowane" i „niebezpieczne" to różne pytania.** `external_exposure` jest osobną
oceną: postawa może być w pełni zgodna z decyzją Foundera **i** mieć otwarte mutacje
produkcyjne. Jedna ocena nie może odpowiadać na oba.

## 5. Obserwowalność tej postawy była ślepa

Przy okazji wyszło, że `ops-observer` raportował ją błędnie od dawna — trzy niezależne
usterki w jednym pliku: pinowanie schematu `v1` równością (producent emituje v2),
słownik trybów `safe/execute_once/unsafe` (nikt tego nie produkuje od v1) i komunikat
`control_safety_contract_invalid`, który nie nazywał żadnej ze stron.

Zmierzone przed naprawą: `observation_ok: false, mode: "unknown",
queue_consumers_enabled: false` przy Control faktycznie w `execute_full`. **Nie brak
danych — konkretna nieprawda.** Testy przechodziły, bo asercjonowały ten sam martwy
kontrakt co kod.

## 6. Co pozostaje otwarte

Sytuacja jest teraz zapytywalna i obserwowana z dwóch niezależnych stron (trasa
platformy i sonda `autonom`). Nadal jednak **nic nie broni** postawy: nakładka może ją
nadpisać, a jedyną konsekwencją jest ocena `overridden_outside_decision`.

Rozstrzygnięcia wymaga pytanie, którego nie rozstrzyga inżynieria: czy nakładka
bezpieczna ma prawo nadpisać zapisaną decyzję Foundera. Argument za: bezpieczeństwo
musi mieć wyłącznik ostateczny. Argument przeciw: decyzja oznaczona `implemented`,
która nie obowiązuje, jest gorsza niż brak decyzji, bo fałszywie uspokaja.

Trzecia droga: nakładka zachowuje prawo weta, ale **musi je odnotować** — zapis
w tickecie, że postawa została nadpisana, z podaniem nakładki i godziny. Wtedy
`decision_status` przestaje kłamać, a `authority_source` ma co pokazać.
