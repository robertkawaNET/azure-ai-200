# Moduł 03: Sidecary w Azure App Service (kontener główny i pomocniczy, localhost, wspólny /home, diagnoza)

Moduł Microsoft Learn: **Run AI apps with sidecars on Azure App Service** (slug `deploy-sidecar-azure-app-service`) - https://learn.microsoft.com/en-us/training/modules/deploy-sidecar-azure-app-service/
Ścieżka: Implement container application hosting on Azure (moduł 3 z 3), kurs AI-200T00, egzamin **AI-200** (Azure AI Cloud Developer Associate), domena 1 "Develop containerized solutions on Azure".
Cel: po tym module umiesz ocenić, kiedy sidecar pasuje do aplikacji AI, skonfigurować kontener główny i pomocniczy (role, porty, zmienne, pobieranie obrazów tożsamością zarządzaną), połączyć kontenery przez `localhost` i wspólny katalog `/home` oraz zdiagnozować awarie pobierania obrazu, startu, zasobów i łączności.
Czas: ok. 5-7 h (czytanie + trzy ćwiczenia). Ćwiczenia robisz w Azure Cloud Shell (Bash) albo lokalnie z Azure CLI 2.x. Potrzebujesz subskrypcji **płatnej** (ACR Tasks nie działają na darmowych kredytach). Koszt: rejestr ACR Basic i plan App Service B1 (płatne); szczegóły i sprzątanie w części E.

Język kodu: egzamin AI-200 zakłada programowanie w **Pythonie**, więc kod aplikacji (API i sidecar) jest w Pythonie. W ćwiczeniu I1 dodano blok **Odpowiednik C# (.NET)** dla kontenera głównego (API). Polecenia `az`, specyfikacje JSON i teoria są wspólne dla obu języków. Uwaga: moduł Microsoftu uczy głównie konfiguracji i diagnozy; w oryginalnym laboratorium sidecar to model Phi-3 (ok. 2,7 GB), a tu zastąpiono go lekkim serwerem-atrapą, żeby ćwiczenie było tanie i szybkie.

Znaczniki źródeł (linia `_Źródło: ..._` pod nagłówkiem):
- `[unit N]` - jednostka (unit) modułu Microsoft Learn; N to prefiks pliku w `sources/deploy-sidecar-azure-app-service/` (`6-lab` = README laboratorium, `6-starter` = pliki startowe laboratorium).
- `[moduł kursu: <slug>]` - inny moduł tego samego kursu.
- `[docs: URL]` - oficjalna dokumentacja produktu.
- `[autor]` - dodane przez autora tutoriala; `[autor: test <data>]` - sprawdzone testem w danym dniu; `[autor: niezweryfikowane]` - jeszcze nie sprawdzone w Azure.

W tekście "moduł N" znaczy plik tutoriala `0N-*.md` (moduł 01 = `01-store-manage-containers-azure-container-registry.md`, moduł 02 = `02-deploy-containers-azure-app-service.md`).

---

# CZĘŚĆ A - WZORZEC SIDECAR: CO TO JEST I KIEDY GO UŻYĆ

## A1. Czym jest sidecar w App Service i scenariusz
_Źródło: [unit 1] + [unit 2] + [docs: https://learn.microsoft.com/en-us/azure/app-service/overview-sidecar]_

**Sidecar** (dosłownie "przyczepka" motocykla) to pomocniczy kontener działający w tym samym środowisku co **główny kontener** aplikacji Linux. Sidecary dodają możliwości, np. lokalne wnioskowanie modelu AI (inference = uruchamianie modelu na zapytaniu), zbieranie telemetrii czy cache, bez upychania ich w obrazie głównym. Wspólne środowisko tworzy użyteczne sprzężenie operacyjne, ale wzorzec **nie zastępuje** niezależnie hostowanej usługi. [unit 2]

Scenariusz z modułu: backend konwersacyjnej AI dla wewnętrznego portalu wsparcia. API w Pythonie waliduje żądania, stosuje reguły biznesowe i formatuje odpowiedzi; prompty wysyła do lokalnego serwera modelu przez `localhost` (bez osobnego punktu końcowego dla danych regulowanych). Zespół chce aktualizować obraz serwera modelu **bez przebudowy obrazu API**; API i serwer modelu mają skalować się razem (każda instancja API potrzebuje lokalnego modelu); ruch zewnętrzny dostaje tylko API; oba kontenery potrzebują wspólnego katalogu na tymczasowe artefakty modelu (bez trwałości); obrazy mają być pobierane z Azure Container Registry bez przechowywania haseł. [unit 1]

Cele modułu: wyjaśnić, kiedy sidecar pasuje do ściśle powiązanego komponentu AI; skonfigurować kontenery główny i pomocnicze, porty docelowe, zmienne środowiskowe i uwierzytelnianie do prywatnego rejestru; zaimplementować komunikację przez `localhost` i wymianę plików przez współdzielony `/home`; zdiagnozować awarie pobierania obrazu, startu, zasobów i łączności. [unit 1]

Dostępność sidecarów może się różnić dla App Service Environment (ASE = izolowane, dedykowane środowisko App Service) i chmur narodowych; sprawdź aktualne ograniczenia przed wyborem wzorca. [unit 2] [docs: sidecar-overview]

## A2. Model wykonania: jeden kontener główny i do dziewięciu sidecarów
_Źródło: [unit 2] + [docs: https://learn.microsoft.com/en-us/azure/app-service/overview-sidecar]_

Aplikacja z sidecarami ma **jeden kontener główny** i **do dziewięciu sidecarów** (egzamin je lubi). App Service zarządza wszystkimi jako jedną aplikacją, ale każdy proces zachowuje własny obraz, polecenie startowe i konfigurację. Role kontenerów decydują o routingu zewnętrznym, a nie o ważności procesu:

- Kontener główny ma `isMain: true`. App Service kieruje **zewnętrzny ruch HTTP wyłącznie do niego**.
- Każdy sidecar ma `isMain: false` i jest osiągalny dla pozostałych kontenerów przez wspólną **przestrzeń nazw sieci** (network namespace = wspólny zestaw interfejsów i portów lokalnych).

Cykl życia sidecarów jest wspólny z aplikacją głównej:

- App Service uruchamia i zatrzymuje kontenery jako część tej samej aplikacji,
- przy skalowaniu w poziomie (scale out) każda nowa instancja zawiera kontener główny i wszystkie sidecary,
- przy skalowaniu w dół App Service usuwa cały komplet kontenerów z instancji,
- wdrożenie konfiguracji może zaktualizować obraz sidecara niezależnie od obrazu głównego, ale nowa konfiguracja nadal należy do tej samej aplikacji.

Dla scenariusza AI to zaleta: każda instancja API dostaje lokalny serwer modelu, więc API nie musi wyszukiwać zdalnego punktu końcowego ani radzić sobie z różną liczbą replik serwera modelu. [unit 2]

## A3. Sprzężenie i izolacja: kiedy sidecar, a kiedy osobna usługa
_Źródło: [unit 2]_

Sidecary współdzielą z aplikacją główną **przestrzeń sieci i środowisko**. Kontener może łączyć się z portami innych kontenerów przez `localhost`, więc uruchamiaj tylko **zaufane obrazy** (wspólna granica zaufania). Wszystkie kontenery instancji zużywają zasoby tego samego **planu App Service**: plan nie skaluje serwera modelu niezależnie od API, a model zużywający większość pamięci może pogorszyć responsywność API.

Sidecar pasuje, gdy:

- każda instancja aplikacji głównej potrzebuje dokładnie jednego lokalnego procesu pomocniczego,
- proces startuje, zatrzymuje się, jest wdrażany i skalowany razem z aplikacją,
- komunikacja lokalna daje wymierną korzyść (opóźnienie, granica danych),
- łączne zapotrzebowanie na CPU i pamięć mieści się w wybranym planie,
- obrazy pochodzą z zaufanych źródeł.

Osobna usługa jest zwykle lepsza, gdy:

- wiele aplikacji współdzieli jeden punkt końcowy modelu lub cache,
- serwowanie modelu wymaga GPU lub innej polityki skalowania niż API,
- komponent wymaga osobnej granicy sieciowej lub bezpieczeństwa,
- komponent ma być dostępny, gdy aplikacja główna się restartuje lub skaluje do zera,
- awarie komponentu nie mogą zużywać zasobów aplikacji głównej.

Przykład: kompaktowy model językowy używany przez jedno wewnętrzne API może pasować do sidecara; duży model obsługujący wiele aplikacji i wymagający własnej pojemności lepiej hostować w usłudze zaprojektowanej do wnioskowania modeli.

## A4. Dwa modele konfiguracji: sitecontainers a klasyczny kontener niestandardowy
_Źródło: [unit 2] + [docs: https://learn.microsoft.com/en-us/azure/app-service/overview-sidecar]_

Aplikacje z sidecarami używają modelu **`sitecontainers`**: każdy kontener to zasób `Microsoft.Web/sites/sitecontainers`, a aplikacja ma `LinuxFxVersion=sitecontainers`. Różni się to od klasycznego kontenera niestandardowego, w którym obraz zapisany jest jako `DOCKER|<obraz>`, oraz od starszych konfiguracji wielokontenerowych z Docker Compose. W modelu sidecar definicje kontenerów zawierają rolę główną i pomocniczą, porty docelowe, uwierzytelnianie obrazu i montowanie wolumenów.

Możesz **przekonwertować** istniejącą aplikację Linux z kontenerem niestandardowym do konfiguracji sidecar. Konwersja zmienia sposób przechowywania ustawień kontenera: niektóre klasyczne ustawienia, w tym `WEBSITES_PORT` i `DOCKER_REGISTRY_SERVER_*`, **nie konfigurują** kontenerów w modelu sitecontainers (egzamin je lubi; porównaj z modułem 02, B2). Zaplanuj ustawienia zastępcze przed konwersją aplikacji produkcyjnej.

Różnica ról w JSON:

```json
{ "name": "main-api",     "properties": { "isMain": true  } }
{ "name": "model-server", "properties": { "isMain": false } }
```

Dokładnie jeden kontener ma `isMain: true`; pozostałe `false`.

## A5. Planowanie pojemności i granic awarii
_Źródło: [unit 2]_

Pojemność planujesz dla **całej instancji**: API, serwer modelu i inne sidecary razem. Szczyt współbieżności i inicjalizacja modelu podnoszą zużycie ponad pomiary stanu ustalonego. Procedura:

1. zmierz pamięć potrzebną do załadowania modelu przed ruchem,
2. zmierz CPU i pamięć aplikacji głównej i sidecara przy reprezentatywnych równoległych żądaniach,
3. uwzględnij nakładanie się startów (App Service może startować kontenery podczas wdrożenia lub skalowania),
4. wybierz plan z pojemnością na łączny szczyt plus zapas operacyjny,
5. po wdrożeniu przejrzyj metryki App Service i przetestuj skalowanie.

Jeśli pomiar pokazuje, że serwowanie modelu dominuje pojemność lub wymaga innej reguły skalowania, oddziel serwer modelu od aplikacji webowej. Dodatkowa granica usługi kosztuje pracą sieciową i operacyjną, ale zapobiega rywalizacji o zasoby z API. SKU, pojemność i ceny planów zmieniają się; sprawdź https://azure.microsoft.com/pricing/details/app-service/linux/. [unit 2]

---

# CZĘŚĆ B - KONFIGURACJA KONTENERA GŁÓWNEGO I SIDECARÓW

## B1. Tworzenie aplikacji obsługującej sidecary
_Źródło: [unit 3] + [docs: https://learn.microsoft.com/en-us/cli/azure/webapp/sitecontainers]_

Przy tworzeniu aplikacji Linux włącz obsługę sidecarów opcją `--sitecontainers-app`; definicje kontenerów tworzysz osobno.

```azurecli
az webapp create \
  --name <app-name> \
  --resource-group <resource-group> \
  --plan <app-service-plan> \
  --sitecontainers-app
```

Istniejącą aplikację z kontenerem niestandardowym możesz przekonwertować:

```azurecli
az webapp sitecontainers convert \
  --mode sitecontainers \
  --name <app-name> \
  --resource-group <resource-group>
```

Przed konwersją zrób inwentaryzację obrazu, portu, rejestru, polecenia startowego i zmiennych, bo `DOCKER_REGISTRY_SERVER_*` i `WEBSITES_PORT` nie konfigurują site containers. Zweryfikuj wynik w środowisku nieprodukcyjnym, zanim skierujesz ruch produkcyjny. Polecenia grupy `az webapp sitecontainers` zmieniają się z wersjami Azure CLI; sprawdź aktualną dokumentację (https://learn.microsoft.com/en-us/cli/azure/webapp/sitecontainers). [unit 1]

## B2. Role kontenerów i porty
_Źródło: [unit 3] + [docs: https://learn.microsoft.com/en-us/azure/app-service/overview-sidecar]_

Każdy kontener ma unikalną nazwę i pełne odwołanie do obrazu. Kontener przyjmujący połączenia powinien zadeklarować **port docelowy (target port)**, a każdy nasłuchujący proces musi użyć **unikalnego portu**. Dokładnie jeden kontener ma `isMain: true`, a jego proces musi nasłuchiwać na porcie **80 lub 8080**, bo App Service obsługuje tylko te porty dla zewnętrznego ruchu HTTP (egzamin je lubi). Do komunikacji wewnętrznej sidecary mogą używać dowolnego innego wolnego, unikalnego portu.

```azurecli
az webapp sitecontainers create \
  --name <app-name> \
  --resource-group <resource-group> \
  --container-name main-api \
  --image <registry-name>.azurecr.io/support-api:<tag> \
  --target-port 8080 \
  --is-main

az webapp sitecontainers create \
  --name <app-name> \
  --resource-group <resource-group> \
  --container-name model-server \
  --image <registry-name>.azurecr.io/model-server:<tag> \
  --target-port 11434
```

Pominięcie `--is-main` zostawia `isMain` jako false. Kontenery dzielą przestrzeń sieci, więc **dwa procesy nie mogą zająć tego samego portu**. **Port docelowy to tylko metadane** i nie steruje routingiem App Service: obraz nadal musi uruchomić proces nasłuchujący na zadeklarowanym porcie. Zmiana samych metadanych nie przekonfiguruje procesu w obrazie, a sidecar, który nie przyjmuje połączeń sieciowych, może pominąć tę właściwość.

## B3. Konfiguracja wspólna i specyficzna dla kontenera
_Źródło: [unit 3]_

**Ustawienia aplikacji (app settings) są dostępne dla wszystkich kontenerów.** **Zmienne środowiskowe specyficzne dla kontenera** należą do jednej definicji site container. Dzięki temu unikasz duplikowania wartości wspólnych, a szczegóły implementacji zostają przy procesie, który je posiada.

Dla scenariusza AI główne API dostaje zmienną `MODEL_ENDPOINT` o wartości `http://localhost:11434` (taka sama na każdej instancji), a serwer modelu zmienną wybierającą model lub sterującą startem. W zasobie site container pole `environmentVariables[].value` **wskazuje nazwę ustawienia aplikacji**, a nie literalną wartość. Najpierw utwórz ustawienia źródłowe:

```azurecli
az webapp config appsettings set \
  --name <app-name> \
  --resource-group <resource-group> \
  --settings \
    MODEL_ENDPOINT_VALUE=http://localhost:11434 \
    MODEL_NAME_VALUE=support-model
```

Specyfikacja obu kontenerów w jednej tablicy JSON mapuje ustawienia źródłowe na nazwy zmiennych kontenera:

```json
[
  {
    "name": "main-api",
    "properties": {
      "image": "<registry-name>.azurecr.io/support-api:<tag>",
      "targetPort": "8080",
      "isMain": true,
      "environmentVariables": [
        { "name": "MODEL_ENDPOINT", "value": "MODEL_ENDPOINT_VALUE" }
      ]
    }
  },
  {
    "name": "model-server",
    "properties": {
      "image": "<registry-name>.azurecr.io/model-server:<tag>",
      "targetPort": "11434",
      "isMain": false,
      "environmentVariables": [
        { "name": "MODEL_NAME", "value": "MODEL_NAME_VALUE" }
      ]
    }
  }
]
```

```azurecli
az webapp sitecontainers create \
  --name <app-name> \
  --resource-group <resource-group> \
  --sitecontainers-spec-file ./sitecontainers-spec.json
```

Przy użyciu `--sitecontainers-spec-file` polecenie ignoruje pozostałe argumenty site container. **Nie umieszczaj poświadczeń w pliku specyfikacji**; używaj ustawień App Service, tożsamości zarządzanych lub odwołań do sekretów (moduł 02, C5).

## B4. Uwierzytelnianie pobierania obrazów tożsamością zarządzaną
_Źródło: [unit 3] + [unit 5] + [docs: https://learn.microsoft.com/en-us/azure/app-service/overview-sidecar] + [moduł kursu: deploy-containers-azure-app-service]_

ACR może przechowywać obrazy głównego kontenera i sidecarów. **Tożsamość zarządzana** pozwala App Service uwierzytelnić się w rejestrze bez hasła w konfiguracji; tożsamość potrzebuje uprawnienia do pobierania obrazów. Możesz użyć tożsamości **system-assigned** lub **user-assigned**; user-assigned przydaje się, gdy wiele aplikacji współdzieli kontrolowaną tożsamość pobierania albo gdy tożsamość ma istnieć niezależnie od aplikacji. Definicja kontenera używa wtedy `authType: UserAssigned` i identyfikuje tożsamość przez **client ID**:

```json
{
  "name": "model-server",
  "properties": {
    "image": "<registry-name>.azurecr.io/model-server:<tag>",
    "targetPort": "11434",
    "isMain": false,
    "authType": "UserAssigned",
    "userManagedIdentityClientId": "<identity-client-id>"
  }
}
```

Co trzeba spełnić (egzamin je lubi):

- tożsamość jest **przypisana do aplikacji** i ma rolę pobierania,
- rola zależy od **trybu uprawnień przypisań ról** rejestru: `AcrPull` przy **RBAC Registry Permissions**; `Container Registry Repository Reader` (opcjonalnie z warunkiem na repozytorium) przy **RBAC Registry + ABAC Repository Permissions**,
- rejestr musi akceptować **tokeny z odbiorcą Azure Resource Manager (ARM audience)** dla pobierania tożsamością zarządzaną; sprawdzisz to poleceniem `az acr config authentication-as-arm show` (włączenie: `az acr config authentication-as-arm update --registry <rejestr> --status enabled`); błąd `UNAUTHORIZED` z komunikatem `token validation failed` może oznaczać, że rejestr ich nie akceptuje. [unit 3] [unit 6-lab]
- przypisanie roli może się propagować kilka minut; zaraz po nadaniu dostępu pobranie obrazu może chwilowo zawieść, więc poczekaj, zanim uznasz konfigurację za błędną i zastąpisz ją hasłem,
- przy firewallu rejestru pozwól adresom wychodzącym aplikacji albo użyj prywatnego punktu końcowego rejestru, DNS i integracji App Service z siecią wirtualną; wtedy włącz opcję **Pull image over VNet**. [unit 5]

Dla kontrolowanych wydań używaj **niezmiennych tagów lub digestów** (moduł 01, C1 i A4); mutowalny tag `latest` może wskazywać inny obraz po restartach, co utrudnia rollback i diagnozę. [unit 3]

> Uwaga o labie (unit 6-lab): laboratorium używa jednej tożsamości user-assigned dla obu kontenerów (z rolą AcrPull) i sprawdza przed buildem, czy polityka `authentication-as-arm` rejestru jest włączona. Tekst jednostki 3 dopuszcza także tożsamość system-assigned.

## B5. Weryfikacja wdrożonych definicji
_Źródło: [unit 3] + [unit 5]_

Zanim przetestujesz zachowanie aplikacji, sprawdź, że definicje dotarły do App Service (to oddziela pomyłki wdrożeniowe od awarii runtime):

```azurecli
az webapp sitecontainers list \
  --name <app-name> \
  --resource-group <resource-group> \
  --output table

az webapp sitecontainers show \
  --name <app-name> \
  --resource-group <resource-group> \
  --container-name model-server
```

Potwierdź, że **jedna** definicja jest kontenerem głównym, a każdy port jest unikalny. Porównaj tagi obrazów z zamierzonym wydaniem i sprawdź typ uwierzytelniania, zanim zaczniesz szukać w logach aplikacji.

---

# CZĘŚĆ C - KOMUNIKACJA MIĘDZY KONTENERAMI I WSPÓŁDZIELONE PLIKI

## C1. Połączenie przez wspólną przestrzeń sieci
_Źródło: [unit 4]_

Wszystkie kontenery aplikacji współdzielą jedną przestrzeń nazw sieci. Kontener łączy się z innym przez **`localhost:<port>`**, a nie przez nazwę kontenera ani usługi (egzamin je lubi). W scenariuszu AI główne API wywołuje serwer modelu pod `http://localhost:11434`. Adres jest lokalny dla instancji: po skalowaniu w poziomie każdy kontener główny wywołuje sidecar na **własnej** instancji.

Fragment Pythona z ograniczonym klientem HTTP (bez limitów czasu zawieszony serwer modelu mógłby trzymać żądanie API w nieskończoność):

```python
import asyncio

import httpx

MODEL_ENDPOINT = "http://localhost:11434/v1/chat/completions"

async def request_completion(messages: list[dict[str, str]]) -> dict:
    timeout = httpx.Timeout(10.0, connect=2.0)              # (1)
    async with httpx.AsyncClient(timeout=timeout) as client:
        async with asyncio.timeout(30.0):                   # (2)
            response = await client.post(
                MODEL_ENDPOINT,
                json={"model": "support-model", "messages": messages},
            )
        response.raise_for_status()
        return response.json()
```

Objaśnienie: (1) krótki limit połączenia (2 s) wykrywa proces, który nie nasłuchuje; ogólny limit HTTPX (10 s) ogranicza bezczynność przy odczycie, zapisie i puli połączeń; (2) `asyncio.timeout` (Python 3.11+) ustawia 30-sekundowy termin dla całego żądania. Wartości dobierz do zmierzonego startu i wnioskowania, a nie arbitralnie. App Service kieruje ruch zewnętrzny **tylko do kontenera głównego**; port docelowy sidecara nie publikuje go jako osobnego publicznego punktu końcowego. Główna aplikacja pozostaje odpowiedzialna za uwierzytelnianie zewnętrzne, autoryzację, walidację żądań i filtrowanie odpowiedzi.

## C2. Niezawodne żądania lokalne
_Źródło: [unit 4]_

Komunikacja lokalna usuwa routing zewnętrzny, ale nie usuwa awarii procesów rozproszonych: proces modelu może się zrestartować, wyczerpać pamięć, odrzucić żądanie lub przekroczyć budżet czasu API. Traktuj wywołanie lokalne jak zależność podatną na błąd. Rozsądna polityka:

- krótki limit połączenia (wykrycie niedostępnego sidecara),
- termin na poziomie aplikacji dopasowany do opóźnienia dla klienta,
- anulowanie, gdy kończy się oryginalne żądanie API,
- **ograniczony retry tylko gdy operacja jest bezpieczna do powtórzenia** (idempotentna),
- czytelne mapowanie błędów (tymczasowa niedostępność vs niepoprawne dane),
- identyfikatory korelacji w logach obu kontenerów.

Nie ponawiaj każdego błędu: drugie żądanie może zdublować pracę, gdy serwer modelu wykonał pierwsze, a odpowiedź zginęła; retry zwiększa też presję na zasoby podczas awarii sidecara. Ważna jest też **gotowość (readiness)**: kontener główny może wystartować, zanim model się załaduje. API może zwracać tymczasową niedostępność, dopóki lekkie żądanie zdrowia sidecara nie zakończy się sukcesem.

## C3. Współdzielony wolumen `/home`
_Źródło: [unit 4] + [unit 6-lab]_

Domyślnie App Service montuje **ten sam wolumen `/home`** w kontenerze głównym i w każdym sidecarze. Kontenery mogą wymieniać pliki przez dedykowany katalog, np. `/home/models`, bez tablicy `volumeMounts` w definicjach. To najprostsza opcja, gdy wszystkie kontenery mogą używać tej samej ścieżki. Wbudowane montowanie działa, **chyba że** aplikacja ma `WEBSITES_ENABLE_APP_SERVICE_STORAGE=false`; to ustawienie decyduje też o tym, czy dane w `/home` dla niestandardowych kontenerów Linux przeżywają restarty i są dzielone między instancjami (moduł 02, B3). Do tymczasowej koordynacji w obrębie jednej instancji traktuj pliki jako odtwarzalne, nawet jeśli magazyn App Service jest włączony.

W scenariuszu `MANIFEST_PATH` serwera modelu i `MODEL_INFO_PATH` głównego API wskazują `/home/models/manifest.json`. Serwer modelu zapisuje gotowy artefakt tak, żeby API nie przeczytało pliku zapisanego w połowie:

```python
from pathlib import Path

SHARED_DIRECTORY = Path("/home/models")
SHARED_DIRECTORY.mkdir(parents=True, exist_ok=True)

temporary_manifest = SHARED_DIRECTORY / "manifest.json.tmp"      # (1)
completed_manifest = SHARED_DIRECTORY / "manifest.json"
temporary_manifest.write_text('{"model": "support-model"}', encoding="utf-8")
temporary_manifest.replace(completed_manifest)                   # (2)
```

Objaśnienie: (1) zapis do nazwy tymczasowej; (2) operacja zmiany nazwy kończy zapis, więc czytelnik widzi tylko kompletny plik. Główne API czyta `/home/models/manifest.json`.

## C4. Unikaj niepotrzebnych własnych montowań
_Źródło: [unit 4] + [unit 5]_

Domyślny wolumen `/home` spełnia potrzebę wymiany plików, więc **nie dodawaj wpisów `volumeMounts`** do definicji głównej i sidecarów. Jawne montowania zmieniają widok systemu plików kontenera i mogą uniemożliwić start aplikacji lub endpointu SCM, gdy konfiguracja montowania koliduje ze środowiskiem. Jeśli istniejące wdrożenie przestaje działać po dodaniu własnych montowań, usuń `volumeMounts` i przywróć ostatnią działającą definicję kontenerów, zanim zaczniesz szukać innych przyczyn. (Dokumentacja produktu dopuszcza własne montowania wolumenów dla każdego kontenera [docs: sidecar-overview]; moduł zaleca prostotę tam, gdzie wystarcza domyślny `/home`.)

## C5. Wybór granicy magazynu
_Źródło: [unit 4]_

Współdzielony `/home` nadaje się do plików z operacyjnego przepływu aplikacji: **plików tymczasowych** tworzonych i zużywanych na tej samej instancji, **wygenerowanej konfiguracji** odświeżanej przez sidecar i czytanej przez aplikację, **metadanych modelu możliwych do odtworzenia** z trwałego źródła i **krótkotrwałych artefaktów pośrednich**. Nie traktuj go jako rejestru modeli, magazynu audytowego ani niezależnego trwałego systemu rekordów. Gdy dane potrzebują jawnej, trwałej granicy poza aplikacją, użyj **Azure Blob Storage** lub **Azure Files**; gdy wiele aplikacji potrzebuje skoordynowanego dostępu, spójności lub niezależnego skalowania, użyj zarządzanej usługi danych. Wymiana plików wymaga też protokołu własności: zapis przez nazwę tymczasową i zmianę nazwy albo publikacja małego manifestu dopiero po przygotowaniu wszystkich artefaktów (patrz C3).

## C6. Walidacja komunikacji i dostępu do plików
_Źródło: [unit 4]_

Testuj te same ścieżki i porty, których używa kod aplikacji. Poprawna odpowiedź zewnętrzna z kontenera głównego **nie dowodzi**, że sidecar jest gotowy; testuj zależność lokalną przez operację zdrowia aplikacji lub endpoint diagnostyczny, który nie ujawnia wrażliwych szczegółów modelu. Kolejność weryfikacji:

1. każdy kontener ma unikalny port docelowy,
2. logi procesu modelu pokazują nasłuch na oczekiwanym porcie,
3. lekkie wywołanie zdrowia modelu z kontenera głównego,
4. reprezentatywne żądanie wnioskowania z zapisem opóźnienia,
5. zapis artefaktu testowego z kontenera piszącego,
6. odczyt artefaktu przez kontener konsumenta (ścieżka współdzielona),
7. restart aplikacji i potwierdzenie, że aplikacja odtwarza wymagane dane tymczasowe.

Nie wystawiaj ogólnego proxy do sidecara; wąska operacja zdrowia lub wnioskowania zachowuje API jako zewnętrzną granicę bezpieczeństwa.

---

# CZĘŚĆ D - DIAGNOZA AWARII SIDECARA

## D1. Przepływ "od objawu"
_Źródło: [unit 5]_

Awaria sidecara często wygląda jak błąd aplikacji głównej, bo kontenery działają jako jedna aplikacja. Rozdziel granice: wdrożenie, pobranie obrazu, start procesu, presja zasobów, komunikacja lokalna. Zaczynaj od objawu widocznego z zewnątrz i śledź żądanie w stronę sidecara; **nie zmieniaj kilku ustawień naraz**, bo utrudnia to wskazanie przyczyny. Zapisuj czas awarii, instancję, identyfikator korelacji i nazwę kontenera. Kolejność:

1. potwierdź, że oczekiwane definicje kontenerów dotarły do App Service,
2. sprawdź, czy obraz został pobrany,
3. potwierdź, że proces sidecara startuje i pozostaje aktywny,
4. przejrzyj CPU i pamięć pod kątem wspólnej presji zasobów,
5. przetestuj połączenie `localhost` z aplikacji głównej,
6. zwaliduj żądanie i odpowiedź na warstwie aplikacji sidecara.

Polecenia: `az webapp sitecontainers list` (wszystkie definicje: wykryjesz brakujący sidecar, nieoczekiwany tag lub wiele ról głównych), `az webapp sitecontainers show --container-name <nazwa>` (pełna definicja: obraz, port, tożsamość, zmienne, polecenie startowe, montowania) i `az webapp sitecontainers status --container-name <nazwa>` (czy platforma raportuje kontener jako działający):

```azurecli
az webapp sitecontainers status \
  --name <app-name> \
  --resource-group <resource-group> \
  --container-name model-server
```

Inspekcja konfiguracji jest ważna: dziennik runtime nie pokaże ustawienia, które nigdy nie dotarło do zasobu. Eksportowane logi mogą zawierać nazwy obrazów, szczegóły środowiska lub dane aplikacji, więc obchodź się z nimi zgodnie z polityką ochrony danych.

## D2. Awarie pobrania obrazu i startu
_Źródło: [unit 5]_

**Awaria pobrania obrazu** zdarza się, zanim wystartuje proces aplikacji: zwykle zły adres obrazu, brak autoryzacji w rejestrze lub ograniczenie sieci rejestru. **Awaria startu** zdarza się po pobraniu obrazu, przy próbie uruchomienia skonfigurowanego polecenia. Dla obrazu z ACR sprawdź:

- serwer logowania, repozytorium i tag zgadzają się z istniejącym obrazem,
- tożsamość zarządzana jest przypisana do aplikacji,
- tożsamość ma `AcrPull` (RBAC Registry Permissions) albo `Container Registry Repository Reader` (RBAC Registry + ABAC Repository Permissions),
- definicja kontenera ma zamierzony typ uwierzytelniania i client ID tożsamości,
- rejestr akceptuje tokeny z odbiorcą ARM,
- reguły firewalla rejestru lub prywatny punkt końcowy, DNS i integracja z siecią wirtualną dają ścieżkę pobrania,
- opcja **Pull image over VNet** jest włączona, gdy obraz ma być pobierany przez integrację z siecią wirtualną.

Propagacja ról trwa: po nadaniu roli poczekaj i ponów próbę, zanim zastąpisz poprawne ustawienia tożsamości zapisanymi poświadczeniami. Logi jednego kontenera:

```azurecli
az webapp sitecontainers log \
  --name <app-name> \
  --resource-group <resource-group> \
  --container-name model-server
```

Gdy obraz się pobiera, ale sidecar nie pozostaje dostępny, sprawdź: punkt wejścia obrazu i polecenie startowe; czy główny proces pozostaje na pierwszym planie (nie kończy się po uruchomieniu procesu potomnego); wymagane zmienne i ścieżki plików; architekturę procesora i zgodność obrazu bazowego; skonfigurowany port docelowy kontra port, na którym proces faktycznie nasłuchuje; dostępność i uprawnienia plików modelu przy inicjalizacji. Kod wyjścia lub ślad stosu zwykle wskazuje obraz lub konfigurację; powtarzane starty bez błędu aplikacji mogą oznaczać presję pamięci przy ładowaniu modelu.

## D3. Presja zasobów
_Źródło: [unit 5]_

Kontener główny i sidecary dzielą zasoby planu na instancji. Duże zużycie pamięci przez model może powodować restarty lub spowolnienie któregokolwiek procesu. Skalowanie API w poziomie tworzy nowy sidecar na każdej nowej instancji, więc inicjalizacja zwiększa łączne zapotrzebowanie planu. Przejrzyj CPU, pamięć, czas odpowiedzi, kolejkę HTTP i sygnały restartów w czasie awarii i porównaj z logami sidecara (ładowanie modelu, zrywy żądań); pojedyncza metryka rzadko dowodzi przyczyny, ale zgodne znaczniki czasu odróżniają presję pojemności od złego portu lub niepoprawnego żądania. Zmierz: pamięć przed i po załadowaniu modelu, szczyt przy współbieżnych żądaniach, CPU i czas odpowiedzi przy docelowej współbieżności, czas startu przy wdrożeniu i skalowaniu oraz zachowanie przy przekroczeniu współbieżności serwera modelu. Jeśli zapotrzebowanie przekracza bezpieczną pojemność planu: zmniejsz współbieżność, wybierz mniejszy model, przeskaluj plan albo hostuj wnioskowanie osobno. **Nie używaj skalowania, żeby ukryć awarię procesu, brakujące ustawienie lub zły tag obrazu.**

## D4. Diagnoza łączności lokalnej
_Źródło: [unit 5]_

Interpretacja objawów (egzamin je lubi): **odmowa połączenia (connection refused)** oznacza, że w tej chwili żaden proces nie przyjął połączenia na żądanym porcie lokalnym; **timeout** może oznaczać wolny start, presję zasobów, zablokowaną pracę aplikacji lub żądanie przekraczające budżet przetwarzania; **odpowiedź błędu HTTP** oznacza, że połączenie sieciowe się udało i badać trzeba format żądania, autoryzację lub zachowanie serwera modelu. Kolejność sprawdzania:

1. kod aplikacji używa `localhost`, a nie nazwy kontenera sidecara,
2. port klienta zgadza się z portem docelowym sidecara,
3. żaden inny kontener nie używa tego samego portu,
4. logi sidecara pokazują adres i port nasłuchu,
5. lekkie żądanie zdrowia z kodu aplikacji głównej,
6. reprezentatywne żądanie wnioskowania z jawnym limitem czasu.

```python
import httpx

async def check_model_health() -> str:
    try:
        async with httpx.AsyncClient(timeout=2.0) as client:
            response = await client.get("http://localhost:11434/health")
            response.raise_for_status()
            return "ready"
    except httpx.ConnectError:
        return "connect-error"
    except httpx.TimeoutException:
        return "timeout"
    except httpx.HTTPStatusError as error:
        return f"http-{error.response.status_code}"
```

`ConnectError` oznacza, że HTTPX nie nawiązał połączenia (przyczyną może być odmowa połączenia, problem z rozpoznaniem nazwy lub inny błąd sieci). Nie zwracaj szczegółowych błędów wewnętrznych w publicznym endpoincie zdrowia: zapisz kategorię diagnostyczną z identyfikatorem korelacji w logu, a zewnętrznym wywołującym zwróć ograniczony wynik gotowości.

## D5. Awarie współdzielonych plików
_Źródło: [unit 5]_

Problemy z plikami wynikają zwykle z różnych ścieżek aplikacji, uprawnień, jawnej konfiguracji montowań lub błędnego założenia trwałości. Obie aplikacje powinny rozwiązywać plik współdzielony do tej samej ścieżki w `/home`. Sprawdź:

- `WEBSITES_ENABLE_APP_SERVICE_STORAGE` **nie jest** ustawione na `false`, gdy aplikacje polegają na współdzielonym `/home`,
- obie aplikacje używają tej samej ścieżki, np. `/home/models/manifest.json`,
- przy jawnych montowaniach: usuń `volumeMounts` z obu definicji i przywróć domyślny `/home`, zanim zbadasz inne przyczyny,
- użytkownik i uprawnienia plików w każdym obrazie,
- mały plik diagnostyczny zapisany przez piszącego i odczytany przez konsumenta,
- restart aplikacji (czy wymagane pliki da się odtworzyć),
- czy skalowanie w poziomie nie zakłada, że lokalne pliki pojawią się na każdej instancji.

Trwałe artefakty przenieś do Azure Storage.

## D6. Diagnostyka platformowa i zapis incydentu
_Źródło: [unit 5]_

Logi kontenerów dają dowody na poziomie procesu, a Azure Monitor i doświadczenie **Diagnose and solve problems** w App Service dają kontekst platformy. Użyj obu źródeł, gdy awaria obejmuje wdrożenie, restarty, limity zasobów lub przerywaną dostępność; Log Analytics ułatwia korelację zdarzeń w dłuższym oknie, jeśli skonfigurowano diagnostykę (moduł 02, D4). Pełny zapis incydentu obejmuje: aplikację, slot, instancję i nazwę kontenera; wdrożony tag lub digest obrazu; pierwszy i ostatni znacznik czasu awarii; istotne komunikaty logów i informacje o zakończeniu procesu; zachowanie CPU i pamięci wokół zdarzenia; kategorię błędu klienta i identyfikator korelacji; zmianę konfiguracji lub wydanie poprzedzające awarię. Po poprawce powtórz oryginalne żądanie pod reprezentatywnym obciążeniem, żeby udowodnić, że naprawiono granicę, która zawiodła.

---

# CZĘŚĆ E - ĆWICZENIA

Przed ćwiczeniami przeczytaj części A-D. Układ ćwiczeń: Gdzie / Plik, kod lub kroki, Koszt, Zadania, Sprawdź, Sprzątanie. I1 odpowiada laboratorium z modułu (unit 6), ale zamiast modelu Phi-3 (ok. 2,7 GB, plan P2v3 w laboratorium [unit 6-starter]) używa małego serwera-atrapy modelu i podanych w całości aplikacji; I2 i I3 są autorskie i korzystają z zasobów z I1 (zmienne `RG`, `ACR_NAME`, `APP_NAME`, `CLIENT_ID`). Wszystkie ćwiczenia tworzą płatne zasoby; wykonaj Sprzątanie z ostatniego ćwiczenia, które robisz.

> Niesprawdzone: dokładne ceny rejestru ACR Basic i planu App Service B1 (Linux) za dzień - zweryfikuj w https://azure.microsoft.com/pricing/details/container-registry/ i https://azure.microsoft.com/pricing/details/app-service/linux/. Czy plan B1 obsługuje sidecary w Twoim regionie - [autor: niezweryfikowane]; laboratorium Microsoftu używa P2v3.

## I1. Aplikacja z kontenerem głównym i sidecarem modelu, pobieranie obrazów tożsamością zarządzaną - Python i C#
_Źródło: [unit 3] + [unit 6] + [unit 6-lab] + [unit 6-starter] + [autor: niezweryfikowane]_
(rozwinięte w `03-deploy-sidecar-azure-app-service.md` §B1-§B5, §C1, §C3)

**Gdzie / Plik:** Cloud Shell (Bash), katalog `sidecar-lab` z podkatalogami `model-server/` i `api/`. Pliki: `sidecar-lab/model-server/model_server.py`, `sidecar-lab/model-server/Dockerfile`, `sidecar-lab/api/app.py`, `sidecar-lab/api/requirements.txt`, `sidecar-lab/api/Dockerfile` oraz `sidecar-lab/sitecontainers-spec.json` (tworzony poleceniem w kroku 3). Wymaga subskrypcji płatnej.

Krok 1 - sidecar: prosty serwer HTTP (tylko biblioteka standardowa Pythona) udający serwer modelu:

```python
# sidecar-lab/model-server/model_server.py
import json
import os
from http.server import BaseHTTPRequestHandler, HTTPServer
from pathlib import Path

PORT = int(os.getenv("PORT", "11434"))                                      # (1)
MANIFEST = Path(os.getenv("MANIFEST_PATH", "/home/models/manifest.json"))    # (2)


class Handler(BaseHTTPRequestHandler):
    def _send(self, code, payload):
        body = json.dumps(payload).encode("utf-8")
        self.send_response(code)
        self.send_header("Content-Type", "application/json")
        self.send_header("Content-Length", str(len(body)))
        self.end_headers()
        self.wfile.write(body)

    def do_GET(self):
        if self.path == "/health":                                           # (3)
            self._send(200, {"status": "ready"})
        else:
            self._send(404, {"error": "not found"})

    def do_POST(self):
        if self.path != "/v1/chat/completions":
            self._send(404, {"error": "not found"})
            return
        length = int(self.headers.get("Content-Length", 0))
        data = json.loads(self.rfile.read(length) or b"{}")
        messages = data.get("messages") or [{}]
        self._send(200, {"model": "stub-model", "reply": "echo: " + messages[-1].get("content", "")})   # (4)


def write_manifest():
    MANIFEST.parent.mkdir(parents=True, exist_ok=True)
    tmp = MANIFEST.with_suffix(".json.tmp")                                  # (5)
    tmp.write_text(json.dumps({"model": "stub-model", "state": "ready"}), encoding="utf-8")
    tmp.replace(MANIFEST)                                                    # (6)


if __name__ == "__main__":
    write_manifest()
    print(f"model server listening on 0.0.0.0:{PORT}", flush=True)
    HTTPServer(("0.0.0.0", PORT), Handler).serve_forever()
```

Objaśnienie: (1) port sidecara 11434 (jak w module); (2) ścieżka manifestu we współdzielonym `/home`; (3) endpoint gotowości; (4) "odpowiedź modelu" - atrapa odsyła treść; (5)-(6) zapis przez plik tymczasowy i zmianę nazwy (patrz C3).

```dockerfile
# sidecar-lab/model-server/Dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY model_server.py .
EXPOSE 11434
CMD ["python", "-u", "model_server.py"]
```

Krok 2 - kontener główny, API w Pythonie (nasłuch na **8080**, wywołuje sidecar przez `localhost`):

```python
# sidecar-lab/api/app.py
import json
import os
from pathlib import Path

import httpx
from flask import Flask, jsonify, request

app = Flask(__name__)

MODEL_ENDPOINT = os.getenv("MODEL_ENDPOINT", "http://localhost:11434")       # (1)
MODEL_INFO_PATH = Path(os.getenv("MODEL_INFO_PATH", "/home/models/manifest.json"))   # (2)
TIMEOUT = httpx.Timeout(10.0, connect=2.0)                                    # (3)


@app.route("/health/ready")
def ready():
    try:
        r = httpx.get(MODEL_ENDPOINT + "/health", timeout=2.0)
        r.raise_for_status()
        return jsonify(status="ready")
    except httpx.ConnectError:
        return jsonify(status="unavailable", reason="connect-error"), 503     # (4)
    except httpx.TimeoutException:
        return jsonify(status="unavailable", reason="timeout"), 503
    except httpx.HTTPStatusError as e:
        return jsonify(status="unavailable", reason=f"http-{e.response.status_code}"), 503


@app.route("/chat", methods=["POST"])
def chat():
    message = (request.get_json(silent=True) or {}).get("message", "")
    try:
        r = httpx.post(
            MODEL_ENDPOINT + "/v1/chat/completions",
            json={"model": "stub-model", "messages": [{"role": "user", "content": message}]},
            timeout=TIMEOUT,
        )
        r.raise_for_status()
        return jsonify(r.json())
    except httpx.HTTPError as e:
        return jsonify(error="model sidecar unavailable", detail=type(e).__name__), 503


@app.route("/model-info")
def model_info():
    if not MODEL_INFO_PATH.exists():
        return jsonify(error="manifest not found"), 404                       # (5)
    return jsonify(json.loads(MODEL_INFO_PATH.read_text(encoding="utf-8")))
```

Objaśnienie: (1)-(2) adres sidecara i ścieżka manifestu pochodzą z konfiguracji kontenera (patrz B3); (3) krótki limit połączenia i ogólny limit (patrz C1); (4) rozróżnienie awarii połączenia, timeoutu i błędu HTTP (patrz D4), zwracany ograniczony wynik; (5) brak manifestu daje 404 (np. gdy `/home` nie jest współdzielony).

```text
# sidecar-lab/api/requirements.txt
flask==3.1.3
httpx
gunicorn==22.0.0
```

```dockerfile
# sidecar-lab/api/Dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app.py .
EXPOSE 8080
CMD ["gunicorn", "--bind", "0.0.0.0:8080", "--workers", "2", "--access-logfile", "-", "app:app"]
```

Krok 3 - zasoby, tożsamość, aplikacja i definicje kontenerów:

```bash
az login
az provider register --namespace Microsoft.ContainerRegistry
az provider register --namespace Microsoft.Web
export RG=rg-sidecar-i1
export ACR_NAME=acrsc$RANDOM$RANDOM
export APP_PLAN=plan-sidecar
export APP_NAME=sidecar$RANDOM$RANDOM
export ID_NAME=id-acr-pull
az group create --name $RG --location westeurope
az acr create --resource-group $RG --name $ACR_NAME --sku Basic                              # (1)
az acr config authentication-as-arm show --registry $ACR_NAME                                # (2)
az acr build --registry $ACR_NAME --image model-server:v1 ./model-server                     # (3)
az acr build --registry $ACR_NAME --image chat-api:v1 ./api

az identity create --resource-group $RG --name $ID_NAME                                      # (4)
ID_RES=$(az identity show -g $RG -n $ID_NAME --query id -o tsv)
export CLIENT_ID=$(az identity show -g $RG -n $ID_NAME --query clientId -o tsv)
PRINCIPAL_ID=$(az identity show -g $RG -n $ID_NAME --query principalId -o tsv)
ACR_ID=$(az acr show -g $RG -n $ACR_NAME --query id -o tsv)
az role assignment create --assignee-object-id $PRINCIPAL_ID \
    --assignee-principal-type ServicePrincipal --role AcrPull --scope $ACR_ID                # (5)

az appservice plan create --resource-group $RG --name $APP_PLAN --sku B1 --is-linux
az webapp create --resource-group $RG --plan $APP_PLAN --name $APP_NAME --sitecontainers-app # (6)
az webapp identity assign --resource-group $RG --name $APP_NAME --identities $ID_RES         # (7)

az webapp config appsettings set --resource-group $RG --name $APP_NAME --settings \
    MODEL_ENDPOINT_VALUE=http://localhost:11434 \
    MODEL_INFO_PATH_VALUE=/home/models/manifest.json \
    MANIFEST_PATH_VALUE=/home/models/manifest.json                                           # (8)

cat > sitecontainers-spec.json <<EOF
[
  {
    "name": "chat-api",
    "properties": {
      "image": "$ACR_NAME.azurecr.io/chat-api:v1",
      "targetPort": "8080",
      "isMain": true,
      "authType": "UserAssigned",
      "userManagedIdentityClientId": "$CLIENT_ID",
      "environmentVariables": [
        { "name": "MODEL_ENDPOINT", "value": "MODEL_ENDPOINT_VALUE" },
        { "name": "MODEL_INFO_PATH", "value": "MODEL_INFO_PATH_VALUE" }
      ]
    }
  },
  {
    "name": "model-server",
    "properties": {
      "image": "$ACR_NAME.azurecr.io/model-server:v1",
      "targetPort": "11434",
      "isMain": false,
      "authType": "UserAssigned",
      "userManagedIdentityClientId": "$CLIENT_ID",
      "environmentVariables": [
        { "name": "MANIFEST_PATH", "value": "MANIFEST_PATH_VALUE" }
      ]
    }
  }
]
EOF
az webapp sitecontainers create --name $APP_NAME --resource-group $RG \
    --sitecontainers-spec-file ./sitecontainers-spec.json                                     # (9)
az webapp sitecontainers list --name $APP_NAME --resource-group $RG --output table           # (10)
```

Objaśnienie: (1) rejestr Basic; (2) sprawdź, że polityka akceptacji tokenów ARM jest włączona (jeśli nie: `az acr config authentication-as-arm update --registry $ACR_NAME --status enabled`); (3) obrazy budowane w chmurze (moduł 01); (4) tożsamość user-assigned; (5) rola `AcrPull` (rejestr w trybie RBAC Registry Permissions); (6) aplikacja w modelu sitecontainers, jeszcze bez kontenerów; (7) przypisanie tożsamości do aplikacji; (8) ustawienia źródłowe dla zmiennych kontenerów (patrz B3); (9) utworzenie obu kontenerów jedną specyfikacją; (10) weryfikacja definicji.

Krok 4 - weryfikacja działania (pierwszy start może potrwać kilka minut, bo trzeba pobrać obrazy):

```bash
APP_URL=https://$(az webapp show -g $RG -n $APP_NAME --query defaultHostName -o tsv)
az webapp sitecontainers status --name $APP_NAME --resource-group $RG --container-name model-server
az webapp sitecontainers log    --name $APP_NAME --resource-group $RG --container-name model-server
curl -s $APP_URL/health/ready
curl -s $APP_URL/model-info
curl -s -X POST $APP_URL/chat -H "Content-Type: application/json" -d '{"message":"hello"}'
```

**Odpowiednik C# (.NET).** Kontener główny napisany w ASP.NET Core; sidecar (serwer-atrapa w Pythonie) pozostaje bez zmian, bo kontenery mają osobne obrazy. Różnice: port **8080** jest domyślny dla obrazów ASP.NET Core od .NET 8, więc nic nie trzeba ustawiać (porównaj z modułem 02, B2); do wywołań lokalnych służy `IHttpClientFactory` z nazwanym klientem i limitem czasu; sam `sitecontainers update` podmienia obraz kontenera głównego.

Plik `sidecar-lab/api-dotnet/ChatApi.csproj`:

```xml
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>
</Project>
```

```csharp
// sidecar-lab/api-dotnet/Program.cs
using System.Text.Json;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddHttpClient("model", c =>
{
    c.BaseAddress = new Uri(Environment.GetEnvironmentVariable("MODEL_ENDPOINT") ?? "http://localhost:11434");   // (1)
    c.Timeout = TimeSpan.FromSeconds(10);                                                                         // (2)
});
var app = builder.Build();

string manifestPath = Environment.GetEnvironmentVariable("MODEL_INFO_PATH") ?? "/home/models/manifest.json";     // (3)

app.MapGet("/health/ready", async (IHttpClientFactory f) =>
{
    try
    {
        using var r = await f.CreateClient("model").GetAsync("/health");
        r.EnsureSuccessStatusCode();
        return Results.Ok(new { status = "ready" });
    }
    catch (HttpRequestException)
    {
        return Results.Json(new { status = "unavailable", reason = "connect-or-http-error" }, statusCode: 503);    // (4)
    }
    catch (TaskCanceledException)
    {
        return Results.Json(new { status = "unavailable", reason = "timeout" }, statusCode: 503);
    }
});

app.MapPost("/chat", async (IHttpClientFactory f, ChatRequest req) =>
{
    try
    {
        using var r = await f.CreateClient("model").PostAsJsonAsync("/v1/chat/completions",
            new { model = "stub-model", messages = new[] { new { role = "user", content = req.Message } } });
        r.EnsureSuccessStatusCode();
        return Results.Content(await r.Content.ReadAsStringAsync(), "application/json");
    }
    catch (Exception e) when (e is HttpRequestException or TaskCanceledException)
    {
        return Results.Json(new { error = "model sidecar unavailable", detail = e.GetType().Name }, statusCode: 503);
    }
});

app.MapGet("/model-info", () => File.Exists(manifestPath)
    ? Results.Content(File.ReadAllText(manifestPath), "application/json")
    : Results.NotFound(new { error = "manifest not found" }));                                                    // (5)

app.Run();

record ChatRequest(string Message);
```

```dockerfile
# sidecar-lab/api-dotnet/Dockerfile
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src
COPY ChatApi.csproj .
RUN dotnet restore
COPY . .
RUN dotnet publish -c Release -o /out --no-restore

FROM mcr.microsoft.com/dotnet/aspnet:10.0
WORKDIR /app
COPY --from=build /out .
EXPOSE 8080
CMD ["dotnet", "ChatApi.dll"]
```

```bash
az acr build --registry $ACR_NAME --image chat-api-dotnet:v1 ./api-dotnet
az webapp sitecontainers update --name $APP_NAME --resource-group $RG \
    --container-name chat-api --image $ACR_NAME.azurecr.io/chat-api-dotnet:v1       # (1)
sleep 60; curl -s $APP_URL/health/ready
```

Objaśnienie: (1) zamienia obraz kontenera głównego na wersję C#; to samo polecenie z obrazem `chat-api:v1` przywraca Pythona. Obrazy bazowe MCR `dotnet/sdk:10.0` i `dotnet/aspnet:10.0` istnieją (sprawdzone 2026-11-22), a `python:3.12-slim` w Docker Hub. Wersję Pythona sprawdzono lokalnie 2026-11-22: serwer-atrapa i API (Flask test client) działały razem, zwracając `ready`, odpowiedź czatu i manifest; wersję C# tylko skompilowano [autor: test 2026-11-22]. W Azure ćwiczenia nie uruchomiono [autor: niezweryfikowane].

**Koszt:** rejestr ACR Basic (płatny za dzień), plan App Service B1 Linux (płatny za godzinę działania) i buildy ACR Tasks (czas obliczeń); tożsamość zarządzana nie kosztuje (ceny: ostrzeżenie "Niesprawdzone" na początku części E). Przy pracy rzędu godziny koszt jest niewielki, o ile wykonasz Sprzątanie. [unit 6]

**Zadania:** (a) wdróż obie definicje kontenerów i wypisz je poleceniem `list`; (b) sprawdź status i log sidecara oraz `/health/ready`; (c) wyślij żądanie `/chat` i odczytaj `/model-info`; (d) podmień kontener główny na wersję C# i powtórz `/health/ready`; (e) **zepsuj to:** utwórz drugi sidecar z tym samym portem co model: `az webapp sitecontainers create --name $APP_NAME --resource-group $RG --container-name second --image $ACR_NAME.azurecr.io/model-server:v1 --target-port 11434`, odczekaj kilka minut i sprawdź `status` i `log` obu kontenerów; potem usuń go (`az webapp sitecontainers delete --name $APP_NAME --resource-group $RG --container-name second`).

**Sprawdź:** (a) tabela zawiera `chat-api` (główny) i `model-server`; (b) status pokazuje działający kontener, log zawiera `model server listening on 0.0.0.0:11434`, a `/health/ready` zwraca `{"status":"ready"}`; (c) odpowiedź czatu zawiera `"reply":"echo: hello"`, a `/model-info` zwraca `{"model":"stub-model","state":"ready"}`; (d) po zamianie obrazu `/health/ready` nadal zwraca `ready` (C# woła ten sam sidecar przez `localhost`); (e) dwa procesy nie mogą nasłuchiwać na tym samym porcie w jednej przestrzeni sieci, więc jeden z nich nie wystartuje poprawnie (dokładny objaw: błąd adresu zajętego w logu lub restarty kontenera; treść - [autor: niezweryfikowane]); po usunięciu drugiego sidecara `/health/ready` znów działa.

**Sprzątanie:**
```bash
az group delete --name rg-sidecar-i1 --yes --no-wait
az group exists --name rg-sidecar-i1     # po kilku minutach powinno wypisać false
```
Usunięcie grupy usuwa rejestr, plan, aplikację i tożsamość. Zasoby z I1 są używane w I2 i I3, więc wykonaj sprzątanie po ostatnim ćwiczeniu, które robisz. Ostrzeżenie: usunięcie grupy usuwa **wszystkie** zasoby w niej; nie używaj grupy z innymi zasobami. [unit 6-lab]

## I2. Komunikacja przez `localhost` i współdzielony `/home`: kontrola i awarie
_Źródło: [unit 4] + [unit 5] + [autor: niezweryfikowane]_

**Gdzie / Plik:** Cloud Shell (Bash); zasoby z I1 (`RG`, `APP_NAME`); w aplikacji działają kontenery z obrazami `chat-api:v1` (Python) i `model-server:v1`.

```bash
export RG=rg-sidecar-i1
export APP_NAME=$(az webapp list -g $RG --query "[0].name" -o tsv)
APP_URL=https://$(az webapp show -g $RG -n $APP_NAME --query defaultHostName -o tsv)
az webapp sitecontainers update --name $APP_NAME --resource-group $RG \
    --container-name chat-api \
    --image $(az acr list -g $RG --query "[0].loginServer" -o tsv)/chat-api:v1            # (1) wróć do Pythona, jeśli robiłeś C#

# (2) restart: manifest jest odtwarzany przez sidecar
az webapp restart --resource-group $RG --name $APP_NAME
sleep 90; curl -s $APP_URL/model-info

# (3) zepsuj komunikację: zły port w ustawieniu źródłowym
az webapp config appsettings set --resource-group $RG --name $APP_NAME \
    --settings MODEL_ENDPOINT_VALUE=http://localhost:11435
sleep 60; curl -s -i $APP_URL/health/ready | head -n 12
az webapp config appsettings set --resource-group $RG --name $APP_NAME \
    --settings MODEL_ENDPOINT_VALUE=http://localhost:11434                                # (4) napraw
sleep 60; curl -s $APP_URL/health/ready

# (5) zepsuj współdzielony wolumen
az webapp config appsettings set --resource-group $RG --name $APP_NAME \
    --settings WEBSITES_ENABLE_APP_SERVICE_STORAGE=false
sleep 90; curl -s -i $APP_URL/model-info | head -n 12
az webapp config appsettings delete --resource-group $RG --name $APP_NAME \
    --setting-names WEBSITES_ENABLE_APP_SERVICE_STORAGE                                   # (6) przywróć domyślne
sleep 90; curl -s $APP_URL/model-info
```

Objaśnienie: (1) wskazanie obrazu Pythona; (2) restart pokazuje, że sidecar sam odtwarza manifest (dane tymczasowe są odtwarzalne); (3) aplikacja główna ma teraz zły adres, więc `/health/ready` zgłosi problem połączenia (kategoria `connect-error`, kod 503); (4) przywrócenie poprawnego portu; (5) wyłączenie magazynu App Service wyłącza wbudowany współdzielony `/home` (patrz C3, D5); (6) usunięcie ustawienia przywraca domyślne zachowanie.

**Koszt:** brak nowych zasobów; zasoby z I1 nadal naliczają opłaty do czasu Sprzątania.

**Zadania:** (a) zrestartuj aplikację i potwierdź, że `/model-info` znów działa; (b) ustaw zły port w `MODEL_ENDPOINT_VALUE` i zapisz kod odpowiedzi i treść `/health/ready`; (c) napraw port; (d) wyłącz `WEBSITES_ENABLE_APP_SERVICE_STORAGE` i sprawdź `/model-info`; (e) przywróć ustawienie.

**Sprawdź:** (a) `{"model":"stub-model","state":"ready"}`; (b) HTTP 503 z `{"reason":"connect-error","status":"unavailable"}` (port 11435 nikt nie nasłuchuje; ustawienia aplikacji restartują aplikację, więc odczekaj); (c) `{"status":"ready"}`; (d) bez współdzielonego `/home` kontenery nie widzą wspólnego manifestu, więc odpowiedź to 404 `manifest not found` (lub błąd aplikacji, jeśli sidecar nie może zapisać manifestu) - dokładny objaw zależy od wersji platformy, [autor: niezweryfikowane]; (e) po przywróceniu `/model-info` znów zwraca manifest.

**Sprzątanie:** przywróć poprawne ustawienia (`MODEL_ENDPOINT_VALUE=http://localhost:11434`, brak `WEBSITES_ENABLE_APP_SERVICE_STORAGE`). Jeśli to ostatnie ćwiczenie: `az group delete --name rg-sidecar-i1 --yes --no-wait`, potem `az group exists --name rg-sidecar-i1` (po kilku minutach: `false`).

## I3. Diagnoza: nieistniejący tag obrazu sidecara, status i logi
_Źródło: [unit 5] + [docs: https://learn.microsoft.com/en-us/cli/azure/webapp/sitecontainers] + [autor: niezweryfikowane]_

**Gdzie / Plik:** Cloud Shell (Bash); zasoby z I1 i I2 (obrazy `model-server:v1` i `chat-api:v1`).

```bash
export RG=rg-sidecar-i1
export APP_NAME=$(az webapp list -g $RG --query "[0].name" -o tsv)
ACR_LS=$(az acr list -g $RG --query "[0].loginServer" -o tsv)

az webapp sitecontainers show --name $APP_NAME --resource-group $RG --container-name model-server     # (1)
az webapp sitecontainers update --name $APP_NAME --resource-group $RG \
    --container-name model-server --image $ACR_LS/model-server:v999                                  # (2) zepsuj: nieistniejący tag
sleep 120
az webapp sitecontainers status --name $APP_NAME --resource-group $RG --container-name model-server   # (3)
az webapp sitecontainers log    --name $APP_NAME --resource-group $RG --container-name model-server   # (4)
curl -s https://$(az webapp show -g $RG -n $APP_NAME --query defaultHostName -o tsv)/health/ready      # (5)

az webapp sitecontainers update --name $APP_NAME --resource-group $RG \
    --container-name model-server --image $ACR_LS/model-server:v1                                    # (6) napraw
sleep 120
az webapp sitecontainers status --name $APP_NAME --resource-group $RG --container-name model-server
```

Objaśnienie: (1) pełna definicja przed zmianą (obraz, port, tożsamość); (2) obraz, który nie istnieje w rejestrze, symuluje awarię pobrania (D2); (3)-(4) status platformy i log kontenera oddzielają awarię pobrania od awarii startu procesu; (5) skutek widoczny z zewnątrz: aplikacja główna zgłasza niedostępność modelu; (6) przywrócenie poprawnego tagu. Zmieniaj jedną rzecz naraz (D1).

**Koszt:** brak nowych zasobów; zasoby z I1 nadal naliczają opłaty.

**Zadania:** (a) zapisz definicję `model-server` przed zmianą; (b) ustaw nieistniejący tag i zbadaj `status` i `log`; (c) odczytaj `/health/ready` z zewnątrz; (d) przywróć tag `v1` i potwierdź powrót do stanu gotowości; (e) zapisz krótki "zapis incydentu" według D6: kontener, tag, czas, objaw, przyczyna, poprawka.

**Sprawdź:** (a) definicja z `image` ... `model-server:v1`, portem `11434` i `authType: UserAssigned`; (b) status nie pokazuje działającego kontenera, a log lub status wskazują błąd pobrania obrazu (manifest nie znaleziony); dokładna treść - [autor: niezweryfikowane]; (c) `/health/ready` zwraca 503 z kategorią `connect-error` (model nie nasłuchuje na 11434); (d) po poprawce `status` pokazuje działający kontener, a `/health/ready` zwraca `ready`; (e) zapis incydentu zawiera wszystkie pola z D6.

**Sprzątanie:**
```bash
az group delete --name rg-sidecar-i1 --yes --no-wait
az group exists --name rg-sidecar-i1     # po kilku minutach: false
```

---

# CZĘŚĆ F - QUIZ Z MODUŁU MICROSOFTU (5 pytań) + wyjaśnienia

Learn pokazuje pytania, ale nie pokazuje poprawnych odpowiedzi, więc każda odpowiedź poniżej jest **wnioskowana** z treści jednostek.

## F1. Kompaktowy model dla każdej instancji API, który ma startować i skalować się razem z API
-> *Uruchom model w sidecarze App Service obok kontenera głównego API.* (A2, A3). Pułapki: osobna współdzielona usługa ma własne skalowanie i nie startuje razem z API; dodanie procesu do obrazu głównego odbiera niezależną aktualizację obrazów, której chce zespół.
*(odpowiedź wnioskowana - Learn nie pokazuje odpowiedzi; uzasadnienie: unit 2, sekcje "Understand the sidecar execution model" i "Evaluate coupling and isolation")*

## F2. Ruch zewnętrzny ma trafiać tylko do API, a API wywołuje serwer modelu lokalnie
-> *`isMain: true` na API i `isMain: false` na serwerze modelu.* (A2, B2). Pułapki: dwa kontenery główne są niedozwolone (dokładnie jeden `isMain: true`); `WEBSITES_PORT` nie konfiguruje kontenerów w modelu sitecontainers.
*(odpowiedź wnioskowana - Learn nie pokazuje odpowiedzi; uzasadnienie: unit 2, sekcja "Understand the sidecar execution model" i unit 3, sekcja "Define container roles and ports")*

## F3. Prywatne obrazy z ACR (RBAC Registry Permissions) bez haseł w konfiguracji
-> *Przypisz aplikacji tożsamość zarządzaną, nadaj jej `AcrPull` i odwołaj się do tożsamości w każdej prywatnej definicji site container.* (B4). Pułapki: konto admin z `DOCKER_REGISTRY_SERVER_*` przechowuje hasła i nie konfiguruje site containers; publiczne repozytoria nie spełniają wymogu prywatności.
*(odpowiedź wnioskowana - Learn nie pokazuje odpowiedzi; uzasadnienie: unit 3, sekcja "Authenticate private image pulls")*

## F4. Artefakty sidecara mają przetrwać niezależnie od cyklu życia aplikacji i służyć innym aplikacjom jako system rekordów
-> *Azure Blob Storage lub Azure Files jako trwała granica magazynu.* (C5). Pułapki: współdzielony `/home` nie jest niezależnym systemem rekordów; warstwa zapisu obrazu kontenera jest ulotna.
*(odpowiedź wnioskowana - Learn nie pokazuje odpowiedzi; uzasadnienie: unit 4, sekcje "Choose the correct storage boundary")*

## F5. Błąd odmowy połączenia przy wywołaniu `http://localhost:11434` z API
-> *Porównaj port wywołania w API z portem docelowym sidecara i sprawdź, czy proces sidecara nasłuchuje na tym porcie.* (D4). Pułapki: skalowanie planu przed inspekcją konfiguracji nie wskaże przyczyny; zamiana `localhost` na nazwę kontenera jest błędna, bo kontenery komunikują się przez `localhost:<port>`.
*(odpowiedź wnioskowana - Learn nie pokazuje odpowiedzi; uzasadnienie: unit 5, sekcja "Diagnose local connectivity")*

---

# CZĘŚĆ G - PYTANIA KONTROLNE (odpowiedz bez zaglądania)

## G1. Ile kontenerów może mieć aplikacja z sidecarami i który dostaje ruch zewnętrzny?
<details><summary>Odpowiedź</summary>

Jeden kontener główny (`isMain: true`) i do dziewięciu sidecarów (`isMain: false`). Ruch zewnętrzny HTTP trafia tylko do kontenera głównego.
</details>

## G2. Na jakim porcie musi nasłuchiwać kontener główny i jak łączą się kontenery między sobą?
<details><summary>Odpowiedź</summary>

Na porcie 80 lub 8080 (tylko te porty są obsługiwane dla zewnętrznego ruchu HTTP). Kontenery łączą się przez `localhost:<port>` we wspólnej przestrzeni sieci, a każdy proces musi użyć unikalnego portu.
</details>

## G3. Czym jest port docelowy (target port) w definicji site container?
<details><summary>Odpowiedź</summary>

Tylko metadanymi: nie steruje routingiem App Service ani nie zmienia portu procesu w obrazie. Proces w obrazie nadal musi nasłuchiwać na zadeklarowanym porcie; sidecar, który nie przyjmuje połączeń, może pominąć tę właściwość.
</details>

## G4. Jak działają zmienne środowiskowe w definicji site container?
<details><summary>Odpowiedź</summary>

Ustawienia aplikacji są dostępne dla wszystkich kontenerów; zmienna kontenera (`environmentVariables`) jest specyficzna dla jednej definicji, a jej pole `value` wskazuje nazwę ustawienia aplikacji, a nie literalną wartość. Poświadczeń nie umieszcza się w pliku specyfikacji.
</details>

## G5. Co jest potrzebne, żeby aplikacja pobierała prywatne obrazy z ACR tożsamością zarządzaną?
<details><summary>Odpowiedź</summary>

Tożsamość przypisana do aplikacji, rola pobierania w rejestrze (`AcrPull` przy RBAC Registry Permissions; `Container Registry Repository Reader` przy RBAC Registry + ABAC Repository Permissions), `authType` i client ID w definicji kontenera (przy user-assigned) oraz rejestr akceptujący tokeny z odbiorcą ARM (`az acr config authentication-as-arm`). Rola potrzebuje czasu na propagację.
</details>

## G6. Czy sidecary mają osobny wolumen na pliki i jakie są ograniczenia?
<details><summary>Odpowiedź</summary>

Domyślnie wszystkie kontenery współdzielą ten sam wolumen `/home` (o ile `WEBSITES_ENABLE_APP_SERVICE_STORAGE` nie jest `false`), więc `volumeMounts` nie są potrzebne. Pliki są tymczasowe/odtwarzalne w sensie modułu; dane trwałe i wspólne dla wielu aplikacji trzymaj w Azure Blob Storage, Azure Files lub usłudze danych. Zapis przez nazwę tymczasową i zmianę nazwy chroni przed odczytem niepełnego pliku.
</details>

## G7. Kiedy wybrać osobną usługę zamiast sidecara?
<details><summary>Odpowiedź</summary>

Gdy wiele aplikacji współdzieli model lub cache, model wymaga GPU lub innego skalowania niż API, potrzebna jest osobna granica sieci lub bezpieczeństwa, komponent ma działać także przy restarcie aplikacji lub skalowaniu do zera, albo jego awarie nie mogą zużywać zasobów aplikacji głównej.
</details>

## G8. Jak odróżnić odmowę połączenia, timeout i błąd HTTP przy wywołaniu sidecara?
<details><summary>Odpowiedź</summary>

Odmowa połączenia: żaden proces nie nasłuchuje na porcie. Timeout: wolny start, presja zasobów, zablokowana praca lub przekroczony budżet czasu. Błąd HTTP: połączenie się udało, więc badaj format żądania, autoryzację lub serwer modelu. Retry tylko dla idempotentnych operacji i błędów uznanych za przejściowe.
</details>

---

# CZĘŚĆ H - CO DALEJ

## H1. Podsumowanie, następny krok i pokrycie egzaminu
_Źródło: [unit 8] + [docs: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/ai-200]_

**Podsumowanie modułu.** Sidecary w App Service dodają wspierające możliwości AI bez łączenia wszystkich procesów w jednym obrazie. Oceniłeś cykl życia, skalowanie, pojemność i zaufanie przed wyborem wzorca; skonfigurowałeś role kontenerów, unikalne porty docelowe, zmienne środowiskowe i pobieranie obrazów z ACR tożsamością zarządzaną; połączyłeś kontenery przez `localhost` i wymieniałeś pliki przez domyślny wolumen `/home`; użyłeś logów konkretnego kontenera, metryk App Service, inspekcji konfiguracji i testów endpointów do diagnozy pobrania obrazu, startu, zasobów, łączności i plików. [unit 8]

**Co dalej:** to ostatni moduł ścieżki "Implement container application hosting on Azure". Następna ścieżka kursu to "Deploy and manage apps on Azure Container Apps" (moduł `deploy-containers-azure-container-apps`, https://learn.microsoft.com/en-us/training/modules/deploy-containers-azure-container-apps/). Wróć do B4 (tożsamość i ARM audience), gdy konfigurujesz pobieranie obrazów z prywatnego rejestru, oraz do D4-D5 przy diagnozie wieloetapowych awarii.

**Pokrycie egzaminu** - lista umiejętności AI-200 (stan study guide z 2026-10-30), domena 1, część "Implement container application hosting":

| Umiejętność (study guide) | Domena | Status | Gdzie |
|---|---|---|---|
| Build, store, version, and manage container images by using Azure Container Registry | 1 | uczone w module 01 | `01-store-manage-containers-azure-container-registry.md` §A1-A5, §C1-C8 |
| Build and run images by using Azure Container Registry Tasks | 1 | uczone w module 01 | `01-store-manage-containers-azure-container-registry.md` §B1-B6 |
| Deploy containers to Azure App Service, including configuring App Service to supply environment variables and secrets | 1 | uczone w module 02 | `02-deploy-containers-azure-app-service.md` §A1-A6, §B1-B5, §C1-C6; sidecary (rozszerzenie hostingu na wiele kontenerów, zmienne per kontener, tożsamość do pobierania obrazów): tu §A1-A5, §B1-B5 |

Pozostałe umiejętności egzaminu należą do innych modułów kursu; pełny przydział jest w `sources/_exam/skill-map.md`. Żaden punkt study guide nie jest orphanem, więc plik luk egzaminu nie jest potrzebny.

## H2. Uczciwie o egzaminie
_Źródło: [autor]_

Study guide AI-200 nie wymienia sidecarów wprost: umiejętność "Deploy containers to Azure App Service..." jest uczona w module 02, a ten moduł rozszerza ją o hosting wielu kontenerów w jednej aplikacji. Traktuj go jako pogłębienie (rola `isMain`, porty, zmienne per kontener, tożsamość do pobierania obrazów, diagnoza), a nie obowiązkowy temat osobnej pozycji listy umiejętności; mimo to pytania o uwierzytelnianie do ACR, porty App Service (80/8080), wspólny `/home` i diagnozę kontenerów są zgodne z listą. Moduł nie ma kodu SDK. Kod aplikacji w ćwiczeniach jest autorski: serwer-atrapa zastępuje model Phi-3 z laboratorium Microsoftu, więc nie ćwiczy ładowania dużego modelu ani presji pamięci (D3). Przed egzaminem przerób też moduły Container Apps i AKS (domena 1), bo ta ścieżka to tylko pierwsza część domeny.
