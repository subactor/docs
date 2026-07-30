---
{
  "schema": "subactor.doc/v1",
  "id": "docs.plans.component-mirror-pin-reconciliation",
  "version": 1,
  "status": "current",
  "updated": "2026-07-30"
}
---

# Uzgodnienie pinów mirroru komponentów

Plan wykonania zadania #5. Nie da się go domknąć jednostronnie — dwa z sześciu
problemów należą do równoległej sesji, a `pin-dependencies` słusznie odmawia
działania, dopóki ich praca jest niescommitowana.

## Stan zmierzony 2026-07-30

```
komponent                    lock      HEAD      zgodny  brudnych  właściciel
components/agents            9cc3d6c3  0fcb75ec  NIE     1         równoległa sesja
components/contracts         1bb68350  e137de96  NIE     0         równoległa sesja
components/core              1baaa036  78bfbb80  NIE     8         mieszane
components/observability     29752e9b  12e5f256  NIE     0         moje
components/autonomy-lab      8c4b2b63  =         tak     0
components/connectors        fca29fc6  =         tak     0
components/runtime           f25ac9d0  =         tak     0
components/site-generator    4e9e63fb  =         tak     0
components/testkit           9ee3ddfc  =         tak     0
components/views             77ad791c  =         tak     0

10 komponentów | 4 niezgodne piny | 2 brudne drzewa
```

**Dwa z czterech rozjazdów są moje.** `components/core` (`78bfbb8`) i
`components/observability` (`12e5f25`) to commity z tej sesji. Rano zarzuciłem
równoległej sesji, że commituje w podrepo bez aktualizacji locka; zrobiłem dokładnie
to samo cztery razy, zanim to zauważyłem.

## Dlaczego nikt tego nie złapał

`verify-dependencies-lock.mjs` sprawdza piny **tylko z flagą `--worktrees`**. Bez niej
zwraca `ok: true` z `worktrees_checked: false` — czyli sukces bez sprawdzenia.
`npm run dependencies:verify` tę flagę podaje, ale **ten skrypt nie występuje
w żadnym łańcuchu testów**: ani w `test`, ani w `test:meta`.

Mechanizm istniał, działał i nie był wołany. Cztery piny przeleżały cały dzień.

## Co już zrobione

Rozdzielenie dwóch pytań, które były zlepione w jedno:

- **twardy niezmiennik** — czy każdy komponent buduje się z commita zapisanego
  w locku; mirror budujący z niezapisanego commita jest nieodtwarzalny;
- **stan przejściowy** — czy drzewo jest czyste; prawdziwe i fałszywe wielokrotnie
  w ciągu dnia normalnej pracy.

Zlepienie ich oznaczało, że jedyny dostępny tryb był wszystko-albo-nic, więc nie dało
się go wpiąć w łańcuch testów podczas pracy — i nie został wpięty nigdzie.

`verifyDependenciesLock({requireClean})` rozdziela je, `--pins` sprawdza sam
niezmiennik, a brud jest **raportowany, nie ukrywany** (`clean`, `dirty_entries`).
Nowy skrypt: `npm run dependencies:pins`. Cztery testy w
`test/dependencies-lock-modes.test.mjs`, w tym jeden pilnujący, że pominięte drzewo
nigdy nie pojawia się jako zweryfikowane.

## Kolejność, której nie da się skrócić

1. **Równoległa sesja commituje albo odkłada** swoją pracę w `components/agents`
   (1 plik) i `components/core` (8 plików). Bez tego `pin-dependencies` odmawia —
   i słusznie: pin zapisany przy niescommitowanych zmianach opisywałby stan,
   którego nie ma.
2. **`node scripts/pin-dependencies.mjs`** — zapisuje wszystkie cztery piny naraz.
   Jedna operacja, bo lock jest jednym plikiem.
3. **`npm run dependencies:pins`** musi przejść.
4. **Wpiąć `dependencies:pins` w `test:meta`.** Celowo jeszcze nie wpięte: dziś
   byłoby czerwone od pierwszego uruchomienia, a bramka, która zawsze świeci na
   czerwono, przestaje być czytana. Wpięcie ma sens dopiero, gdy krok 3 jest zielony.
5. Sonda `components-pin-drift` w `autonom` obserwuje to niezależnie i **zostaje**
   nawet po wpięciu bramki: bramka pilnuje momentu commita, sonda pilnuje stanu
   między commitami.

## Czego ten plan nie rozwiązuje

Brud w `components/core` (8 plików) to praca w toku, nie dług. Nie należy go pinować
ani ratchetować — właściwym sygnałem jest dryf, tak jak dla `uncommitted-core`.
Krok 1 nie jest „naprawą", tylko naturalnym zakończeniem czyjejś pracy.
