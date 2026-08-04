---
{
  "schema": "subactor.doc/v1",
  "id": "docs.architecture.incidents.2026-08-01-cross-site-deployment",
  "version": 2,
  "status": "current",
  "updated": "2026-08-04"
}
---

# Analiza: deployment autonomicznosc.pl nadpisał subactor.com

## Skutek

Proces publikacji projektu `autonomicznosc-pl` przekazał źródło projektu wraz
z niejednoznacznym celem `/httpdocs` i współdzielonymi credentialami Pleska.
Dla tych credentiali `/httpdocs` oznaczało docroot `subactor.com`, dlatego pliki
innej witryny zostały zapisane na domenie głównej. HTTP 200 po publikacji został
błędnie uznany za sukces mimo niezgodnej treści.

## Co zawiodło warstwa po warstwie

1. `site-resources.json` mieszał katalog źródeł z wyjątkami topologii hostingu.
   Nie istniał odrębny, wersjonowany kontrakt „to źródło → to środowisko → ten
   webspace/docroot → te uchwyty credentiali”.
2. Ticket mógł utrwalić surowe `source_dir`, `domain` i `remote_path` bez
   wspólnej tożsamości bindingu. Pola były poprawne składniowo, lecz nie
   stanowiły jednej atomowej decyzji konfiguracyjnej.
3. `/httpdocs` jest zależne od chrootu credentiala. Sama ścieżka i host
   transportowy `prototypowanie.pl` nie identyfikują domeny docelowej.
4. Twin zwrócił semantyczne `decision=refuse`, ale kontroler sprawdzał sukces
   transportu procesu i wykonał kolejny krok.
5. Konektor nie rewalidował całego celu względem wersjonowanej konfiguracji.
   Grant był związany z planem plików i hostem, lecz nie z trwałym bindingiem
   źródło–domena–webspace–credential.
6. Postflight sprawdzał osiągalność/HTTP 200, a nie identyczność entrypointu.
7. Automatyczne reconciliation produkowało kolejne tickety z tym samym błędnym
   kontraktem, utrwalając problem zamiast go izolować.

## Naprawa

- Źródła pozostały w `platform/config/site-resources.json` i są wybierane tylko
  przez logiczny `source_ref`.
- Cele przeniesiono do
  `platform/config/deployment-bindings/registry.v1.json`, walidowanego schematem.
- Manifest projektu wskazuje `deployment_ref`, nie host, docroot ani credential.
- Payload zawiera `deployment_binding_ref`, wersję i SHA-256 rekordu. Konektor
  ładuje własną kopię registry i wymaga dokładnej zgodności wszystkich pól.
- Konektor ładuje też executorowy `site-resources.json` i wymaga, aby przekazany
  `source_dir` po `realpath` był dokładną ścieżką przypisaną do `source_ref`.
  Samo podszycie dowolnego katalogu pod poprawny identyfikator nie przejdzie.
- W produkcji brak bindingu, zmieniony hash lub choć jedno inne pole celu kończy
  się odmową przed planowaniem i przed leasingiem credentiali.
- Odmowa twina i `ok=false` są twardym końcem procesu.
- Postflight wymaga SHA-256 publicznej treści zgodnego z lokalnym entrypointem.

## Zasady trwałości i przenośności

Registry deploymentów jest niesekretnym artefaktem Git i Artifact Registry.
Może być skopiowane lub pobrane jako konkretna rewizja przez dowolny executor.
Nie zawiera ścieżek lokalnego `.env` ani wartości haseł. `source_ref` jest
rozwiązywany lokalnie przez inventory workspacu, a `credential_refs` przez Vault
w danym miejscu wykonania. Ta sama nazwa bindingu i ten sam digest muszą zostać
potwierdzone po obu stronach granicy Control → connector. Dla wielu executorów
inventory może mieć różne ścieżki fizyczne, ale każda jego wersja jest
walidowanym artefaktem, nie niekontrolowanym zestawem zmiennych środowiskowych.

## Dowód zamknięcia

- `subactor.com` i `autonomicznosc.pl` mają publiczny SHA-256 zgodny z właściwym
  entrypointem źródłowym.
- Reconciliation `PLF-2747` zakończyło się jako `converged` bez blockerów.
- Testy obejmują odmowę obcego docrootu, `/`, niepowiązanych credentiali,
  zmienionego bindingu, źródła z obcej ścieżki, odmowy twina i niezgodnego
  SHA-256.
