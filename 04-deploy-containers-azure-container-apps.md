# Moduł 04: Wdrażanie kontenerów w Azure Container Apps (środowisko, rewizje, sekrety, rejestr prywatny, logi)

Moduł Microsoft Learn: **Deploy containers to Azure Container Apps** - https://learn.microsoft.com/en-us/training/modules/deploy-containers-azure-container-apps/
Cel: po tym module rozumiesz, czym jest środowisko Container Apps, umiesz wdrożyć kontener poleceniami `az containerapp` i plikiem YAML, ustawić zmienne środowiskowe i sekrety, skonfigurować pobieranie obrazu z prywatnego rejestru i sprawdzić wdrożenie w logach, rewizjach i replikach.
Czas: ok. 3-4 h, orientacyjnie (moduł Learn ma ok. 72 min wg katalogu, w tym ćwiczenie ok. 30 min; tutaj dochodzą 3 ćwiczenia z eksperymentami). Ćwiczenia robisz w terminalu z Azure CLI (Cloud Shell albo własny terminal), Python 3.12 do ćwiczenia z SDK. Koszt: ćwiczenia I1 i I2 tworzą płatne zasoby w chmurze (rejestr ACR, środowisko Container Apps, Log Analytics) - szczegóły w polach **Koszt** przy ćwiczeniach.

Źródła: `[unit N]` = jednostka modułu Learn, `[docs: URL]` = dokumentacja produktu, `[autor]` = dodane przez autora; pełna legenda: przewodnik stylu §3. "Moduł N" znaczy plik tutoriala `0N-*.md`.

---

# CZĘŚĆ A - CO TO JEST I JAK TO DZIAŁA

## A1. Czym jest Azure Container Apps i środowisko (environment)
_Źródło: [unit 1] + [unit 2]_

**Azure Container Apps** to usługa do uruchamiania kontenerów (paczek z aplikacją i jej zależnościami) bez zarządzania klastrem Kubernetes. Dostajesz "bezserwerowy" (serverless: nie obsługujesz serwerów sam) silnik kontenerów, wdrożenia oparte na rewizjach, wbudowany ruch przychodzący (ingress) i zarządzanie przez Azure.

Aplikacje działają w **środowisku Container Apps** (Container Apps environment). Środowisko to granica: aplikacje z jednego środowiska współdzielą sieć i miejsce zapisu logów. To nie jest "folder na zasoby" - środowisko wpływa na to, jak ruch wewnętrzny przechodzi między aplikacjami, jak podłączasz logi i jak stosujesz zasady zarządzania. [unit 2]

```
Grupa zasobów (resource group)
 └── Środowisko Container Apps  (wspólna sieć + wspólne logi)
      ├── aplikacja "ai-api"        (ingress external)
      └── aplikacja "przetwarzanie" (ingress internal)
```

Co mówi dokumentacja produktu: środowisko to "secure boundary around one or more container apps and jobs"; apps w jednym środowisku "share the same virtual network and write logs to the same logging destination". Nowe środowiska są domyślnie typu **Workload profiles (v2)**; typ **Consumption-only (v1)** jest dziedziczny (legacy). [docs: https://learn.microsoft.com/azure/container-apps/environment] [docs: https://learn.microsoft.com/azure/container-apps/structure]

(egzamin je lubi: środowisko = granica sieci i logów; jedna aplikacja nie "należy" do dwóch środowisk)

## A2. Tworzenie i sprawdzanie środowiska
_Źródło: [unit 2]_

Rozszerzenie CLI `containerapp` bywa wymagane. Instalacja lub aktualizacja: `az extension add --name containerapp --upgrade`; funkcje w wersji zapoznawczej (preview) włączasz przez `--allow-preview true`.

Środowisko możesz utworzyć jawnie albo pozwolić, by `az containerapp up` utworzyło je za Ciebie. Jawne utworzenie daje kontrolę nad nazwą i cyklem życia, co się przydaje, gdy kilka aplikacji dzieli jedno środowisko.

```azurecli
az group create --name rg-aca-demo --location centralus            # (1)

az containerapp env create \
    --name aca-env-demo \
    --resource-group rg-aca-demo \
    --location centralus                                            # (2)

az containerapp env show \
    --name aca-env-demo \
    --resource-group rg-aca-demo                                    # (3)
```
Objaśnienie: (1) grupa zasobów (resource group, logiczny kontener na zasoby, który usuwasz jednym poleceniem); (2) środowisko; (3) podgląd środowiska - pokazuje, którego środowiska używa aplikacja i pomaga przy problemach z regionem lub konfiguracją.

## A3. Sieć i ingress: external czy internal
_Źródło: [unit 2] + [docs: https://learn.microsoft.com/azure/container-apps/ingress-overview]_

**Ingress** to wbudowany sposób, w jaki ruch HTTP (lub TCP) trafia do aplikacji; nie tworzysz własnego load balancera ani publicznego IP. Dwa rodzaje:

| Opcja CLI | Znaczenie wg modułu | Kiedy |
|---|---|---|
| `--ingress external` | API wywoływane bezpośrednio przez klienta (np. przeglądarkę) [unit 2] | publiczne API |
| `--ingress internal` | usługa wywoływana tylko przez inne usługi w środowisku [unit 2] | backend, procesor w tle |

Dokumentacja dopowiada ważny szczegół: ustawienie ingress opisuje widoczność aplikacji **względem środowiska**, a nie internetu. Adres aplikacji z `internal` ma segment `.internal.`: `<APP>.internal.<ENV_ID>.<REGION>.azurecontainerapps.io`; żądania spoza środowiska dostają 404. Zmiana ustawień ingress dotyczy wszystkich rewizji naraz i **nie tworzy nowej rewizji**. Dla HTTP adres aplikacji jest zawsze wystawiony na porcie 443, a `targetPort` to port, na którym nasłuchuje Twój kontener. [docs: https://learn.microsoft.com/azure/container-apps/ingress-overview] [docs: https://learn.microsoft.com/azure/container-apps/ingress-how-to?pivots=azure-cli#use-additional-tcp-ports]

## A4. Logi w środowisku i dobre praktyki projektowania środowisk
_Źródło: [unit 2]_

Środowisko zwykle integruje się z Azure Monitor (usługa zbierania logów i metryk), dzięki czemu logi kontenerów i logi systemowe znajdziesz w jednym miejscu. Zaplanuj logowanie od początku: pierwsze dochodzenie awarii to zwykle moment, w którym orientujesz się, że logów brakuje. Domyślnie wszystkie aplikacje środowiska wysyłają logi do wspólnego obszaru Log Analytics (obszar roboczy do zapytań o logi); nowe rekordy pojawiają się z opóźnieniem kilku minut, a do podglądu na żywo służy log streaming. [docs: https://learn.microsoft.com/azure/container-apps/environment] 

Dobre praktyki z modułu [unit 2]:

- **Rozdzielaj środowiska według cyklu życia:** osobne dla dev, test i prod, żeby ograniczyć skutki błędów przy sprawdzaniu nowych rewizji.
- **Dopasuj środowiska do granic sieciowych:** usługi, które mają rozmawiać prywatnie, daj do jednego środowiska i używaj `internal` tam, gdzie trzeba.
- **Ujednolić nazewnictwo** grup zasobów, środowisk i aplikacji (narzędzia operacyjne łatwiej je znajdą).
- **Zaplanuj obserwowalność** (gdzie trafiają logi i jak je odpytasz) zanim puścisz ruch produkcyjny.

---

# CZĘŚĆ B - WDRAŻANIE APLIKACJI

## B1. Przygotowanie CLI i dostawców zasobów
_Źródło: [unit 3]_

Przed wdrożeniem potrzebujesz zalogowanego, aktualnego CLI i zarejestrowanych **dostawców zasobów** (resource providers: komponenty Azure, które tworzą dany typ zasobów). Bez rejestracji `Microsoft.App` i `Microsoft.OperationalInsights` tworzenie zasobów się nie uda.

```azurecli
az login
az upgrade
az extension add --name containerapp --upgrade

az provider register --namespace Microsoft.App
az provider register --namespace Microsoft.OperationalInsights
```

> Wersje sprawdzone 2026-11-22 (źródło: `az version` lokalnie, `az extension list-available`): Azure CLI 2.90.0; rozszerzenie `containerapp` w indeksie ma tylko wersję **1.3.0b6** (beta), więc `az extension add` zgłasza "No stable version" i instaluje wersję zapoznawczą. Moduł zakłada, że rozszerzenie działa; przed użyciem sprawdź `az extension list`. [docs: https://learn.microsoft.com/azure/container-apps/get-started]

## B2. Szybkie wdrożenie: `az containerapp up`
_Źródło: [unit 3]_

`az containerapp up` to najszybszy sposób na pierwszą wersję: potrafi utworzyć środowisko i zasoby pomocnicze, więc zmniejsza liczbę decyzji. Dobre do prototypów.

```azurecli
az containerapp up \
    --name my-container-app \
    --resource-group rg-aca-demo \
    --location centralus \
    --environment aca-env-demo \
    --image mcr.microsoft.com/k8se/quickstart:latest \
    --target-port 80 \
    --ingress external \
    --query properties.configuration.ingress.fqdn
```

Polecenie zwraca **FQDN** (fully qualified domain name, pełna nazwa domenowa aplikacji). Ścieżka obrazu musi być zapisana **małymi literami** (np. `mcr.microsoft.com/k8se/quickstart`); wielkie litery dają błędy pobierania obrazu, które wyglądają jak błędy uwierzytelniania. [unit 3]

Dokumentacja: `up` tworzy środowisko, obszar Log Analytics i aplikację, a jeśli któryś z zasobów już istnieje, **użyje go** zamiast tworzyć nowy. [docs: https://learn.microsoft.com/azure/container-apps/get-started]

(egzamin je lubi: `up` = szybko i z domyślnymi decyzjami; `create` = jawnie)

## B3. Jawne wdrożenie: `az containerapp create`
_Źródło: [unit 3]_

Gdy środowisko już istnieje, a aplikację trzeba dopasować do niego i do grupy zasobów, użyj `create`. Łatwiej wtedy wprowadzić spójne wartości domyślne w wielu środowiskach.

```azurecli
az containerapp create \
    --name ai-api \
    --resource-group rg-aca-demo \
    --environment aca-env-demo \
    --image mcr.microsoft.com/k8se/quickstart:latest \
    --ingress external \
    --target-port 80
```

Nazwa aplikacji: małe litery, cyfry lub `-`, zaczyna się od litery, kończy literą lub cyfrą, bez `--`, krócej niż 32 znaki. [docs: https://learn.microsoft.com/cli/azure/containerapp?view=azure-cli-latest]

## B4. Aktualizacja = nowa rewizja (revision)
_Źródło: [unit 3] + [docs: https://learn.microsoft.com/azure/container-apps/revisions]_

**Rewizja** (revision) to niezmienna migawka wersji konfiguracji aplikacji. Gdy zmieniasz np. obraz lub zmienne środowiskowe, powstaje nowa rewizja i możesz ją sprawdzić, zanim przesuniesz na nią cały ruch. [unit 3]

```azurecli
az containerapp update \
    --name ai-api \
    --resource-group rg-aca-demo \
    --image myregistry.azurecr.io/ai-api:v2
```

Reguła, na którą pytają: nie każda zmiana tworzy rewizję.

| Rodzaj zmiany | Gdzie w szablonie | Przykłady | Nowa rewizja? |
|---|---|---|---|
| **Revision-scope** | `properties.template` | obraz i konfiguracja kontenera, zmienne środowiskowe, reguły skalowania, sufiks rewizji | tak |
| **Application-scope** | `properties.configuration` | wartości sekretów, tryb rewizji, ingress (włączenie, podział ruchu, etykiety), poświadczenia rejestru, ustawienia Dapr | nie |

[docs: https://learn.microsoft.com/azure/container-apps/revisions] Domyślnie dostępnych jest 100 nieaktywnych rewizji (próg można zmienić). Nazwa rewizji ma postać `<APP>-<SUFIKS>`; sufiks: małe litery, cyfry lub `-`, zaczyna się od litery, nie ma `--`, maks. 64 znaki. [docs: https://learn.microsoft.com/azure/container-apps/revisions]

> **UWAGA - błąd w materiale Microsoftu** (unit 7, lab): lab pisze, że "Container Apps creates a new revision whenever you change environment variables or secrets" -> zmiana **zmiennych środowiskowych** tworzy rewizję, ale zmiana **wartości sekretu** jej nie tworzy (to zmiana application-scope; aby kontener zobaczył nową wartość, trzeba wdrożyć nową rewizję lub zrestartować istniejącą). Źródło: https://learn.microsoft.com/azure/container-apps/manage-secrets.

## B5. YAML: powtarzalne i recenzowalne wdrożenia
_Źródło: [unit 3]_

Plik YAML (format konfiguracji oparty na wcięciach) trzymasz w repozytorium i zmieniasz przez review, więc maleje "dryf konfiguracji" (różnice między środowiskami). Szczególnie pomaga, gdy usługa AI ma wiele ustawień: flagi funkcji, adresy zależności, skalowanie. Gdy podasz `--yaml`, **pozostałe flagi są ignorowane**, a YAML staje się źródłem prawdy. [unit 3]

```azurecli
az containerapp create --name ai-api --resource-group rg-aca-demo \
    --environment aca-env-demo --yaml ./containerapp.yml

az containerapp update --name ai-api --resource-group rg-aca-demo \
    --yaml ./containerapp.yml
```

Szkielet pliku (struktura potwierdzona w dokumentacji): [docs: https://learn.microsoft.com/azure/container-apps/ingress-how-to?pivots=azure-cli#use-additional-tcp-ports]

```yaml
location: centralus
name: ai-api
type: Microsoft.App/containerApps
properties:
  managedEnvironmentId: <ID środowiska>
  configuration:
    activeRevisionsMode: Single
    ingress:
      external: true
      targetPort: 80
  template:
    containers:
    - name: ai-api
      image: mcr.microsoft.com/k8se/quickstart:latest
    scale:
      minReplicas: 1
      maxReplicas: 1
```

## B6. Inne sposoby wdrażania
_Źródło: [unit 3]_

W prawdziwych projektach przechodzisz od CLI do potoków: **Bicep** (język do infrastruktury jako kod), **GitHub Actions** (ciągłe wdrażanie) albo portal Azure do zmian niezaplanowanych. Wzorce CLI i YAML z tego modułu zostają przydatne: pokazują, które właściwości się zmieniają i gdzie szukać przy awarii.

---

# CZĘŚĆ C - USTAWIENIA URUCHOMIENIOWE: ZMIENNE I SEKRETY

## C1. Zmienne środowiskowe (niewrażliwe ustawienia)
_Źródło: [unit 4]_

**Zmienne środowiskowe** (environment variables) to dobre miejsce na poziomy logowania, flagi funkcji i adresy zależności. Dzięki nim ten sam obraz działa w wielu środowiskach z różnymi wartościami, a limity czasu czy rozmiary partii zmieniasz bez budowania obrazu od nowa.

```azurecli
az containerapp create -n ai-api -g rg-aca-demo \
    --environment aca-env-demo \
    --image myregistry.azurecr.io/ai-api:v1 \
    --ingress external --target-port 8000 \
    --env-vars LOG_LEVEL=info FEATURE_EMBEDDINGS=true

az containerapp update -n ai-api -g rg-aca-demo \
    --set-env-vars LOG_LEVEL=debug
```

`--env-vars` używasz przy tworzeniu. `--set-env-vars` przy aktualizacji **dodaje lub zmienia** wskazane zmienne i nie rusza pozostałych; `--replace-env-vars` zastępuje wszystkie (reszta znika). Gdy nazwa się powtórzy, obowiązuje ostatnia na liście. Zmiana zmiennych po utworzeniu aplikacji zawsze oznacza nową rewizję. [docs: https://learn.microsoft.com/azure/container-apps/environment-variables] [docs: https://learn.microsoft.com/cli/azure/containerapp?view=azure-cli-latest]

## C2. Sekrety i odwołania `secretref:`
_Źródło: [unit 4] + [docs: https://learn.microsoft.com/azure/container-apps/manage-secrets]_

Wartości wrażliwe (klucze API dostawców modeli, hasła do baz, klucze podpisu) trzymasz jako **sekrety** Container Apps. Dzięki temu nie trafiają do obrazu ani do repozytorium, a rotację robisz bez przebudowy obrazu. [unit 4]

```azurecli
az containerapp secret set -n ai-api -g rg-aca-demo \
    --secrets embeddings-api-key="REPLACE_WITH_REAL_VALUE"          # (1)

az containerapp update -n ai-api -g rg-aca-demo \
    --set-env-vars EMBEDDINGS_API_KEY=secretref:embeddings-api-key  # (2)
```
Objaśnienie: (1) sekrety to pary `klucz=wartość`; (2) zmienna środowiskowa o wartości `secretref:<nazwa-sekretu>` dostaje wartość sekretu w czasie działania, bez ujawniania jej w poleceniu. Sekret musi już istnieć.

Moduł wspomina też, że CLI obsługuje odwołanie do **Azure Key Vault** (sejf na sekrety), gdy Key Vault ma być systemem źródłowym, a aplikacja pobiera sekret w czasie działania. [unit 4]

Czego moduł nie mówi, a egzamin lubi [docs: https://learn.microsoft.com/azure/container-apps/manage-secrets]:

- Sekrety są przypisane do **aplikacji**, nie do rewizji; ich zmiana **nie tworzy rewizji**.
- Zaktualizowany lub usunięty sekret nie wpływa automatycznie na działające rewizje: wdróż nową rewizję albo **zrestartuj** istniejącą (`az containerapp revision restart --revision <nazwa>`).
- Przed usunięciem sekretu wdróż rewizję, która już go nie używa, i dezaktywuj rewizje, które się do niego odwołują.
- Brak wbudowanej automatycznej rotacji, wygasania i wersjonowania sekretów w samych sekretach Container Apps (stąd Key Vault). [docs: https://learn.microsoft.com/azure/container-apps/manage-secrets]

## C3. YAML dla ustawień runtime i dobre praktyki
_Źródło: [unit 4]_

Fragment YAML z jedną zmienną jawną i jednym odwołaniem do sekretu. W YAML przechowuj **tylko nazwy** sekretów, nigdy wartości.

```yaml
# Code fragment - focus on environment variables and secret references
properties:
  template:
    containers:
    - name: ai-api
      env:
      - name: LOG_LEVEL
        value: info
      - name: EMBEDDINGS_API_KEY
        secretRef: embeddings-api-key
```

Po edycji stosujesz plik poleceniem `az containerapp update -n ai-api -g rg-aca-demo --yaml ./containerapp.yml`; to polecenie traktuje plik jako pełne źródło konfiguracji. Zwróć uwagę na pisownię: w CLI `secretref:` (małymi literami, w wartości zmiennej), w YAML `secretRef` (pole).

Dobre praktyki [unit 4]: oddzielaj konfigurację od obrazów; preferuj odwołania `secretref:` (wartości nie trafiają do YAML ani historii powłoki); rotuj sekrety niezależnie od obrazów; trzymaj YAML w kontroli wersji.

---

# CZĘŚĆ D - UWIERZYTELNIANIE DO PRYWATNEGO REJESTRU

## D1. Opcje i kompromis
_Źródło: [unit 5]_

Produkcyjne usługi AI zwykle biorą obrazy z **prywatnego rejestru** (np. Azure Container Registry, ACR), bo to ogranicza ryzyko łańcucha dostaw, pozwala skanować podatności i kontrolować, kto pobiera obrazy. Container Apps wspiera uwierzytelnianie nazwą użytkownika i hasłem oraz **tożsamością zarządzaną** (managed identity: tożsamość aplikacji w Entra ID, bez haseł w skryptach), gdy rejestr to ACR.

| Opcja | Zaleta | Wada |
|---|---|---|
| Użytkownik + hasło | działa wszędzie, także w rejestrach bez tożsamości Azure; dobre do szybkiej weryfikacji | długotrwałe poświadczenia do rotowania |
| Tożsamość zarządzana (tylko ACR) | brak haseł; mniej sekretów do rotacji | wymaga przypisania roli (`AcrPull`) |

Uwaga z modułu: konfiguracja rejestru jest osobna od konfiguracji aplikacji. Jeśli aplikacja nie startuje po zmianie rejestru, logi kontenera często pokazują błędy pobierania obrazu wyglądające jak błędy uwierzytelniania lub uprawnień. [unit 5]

## D2. Rejestr: nazwa użytkownika i hasło
_Źródło: [unit 5]_

```azurecli
az containerapp registry set -n ai-api -g rg-aca-demo \
    --server myregistry.azurecr.io \
    --username MyRegistryUsername \
    --password MyRegistryPassword
```

Polecenie wiąże rejestr z aplikacją, więc pobieranie obrazów działa dla kolejnych rewizji. Dla ACR CLI bywa w stanie samo wywnioskować poświadczenia, ale na produkcji wybierz jawny sposób uwierzytelniania, żeby zachowanie było przewidywalne. Poświadczenia rejestru to zmiana application-scope (bez nowej rewizji). [docs: https://learn.microsoft.com/azure/container-apps/revisions]

## D3. Rejestr: tożsamość zarządzana i rola `AcrPull`
_Źródło: [unit 5] + [unit 7] + [docs: https://learn.microsoft.com/azure/container-apps/managed-identity-image-pull]_

```azurecli
az containerapp registry set -n ai-api -g rg-aca-demo \
    --server myregistry.azurecr.io \
    --identity system
```

`--identity system` oznacza tożsamość systemową aplikacji (tworzoną i usuwaną razem z aplikacją). Tożsamość potrzebuje roli **`AcrPull`** na rejestrze (najmniejszy potrzebny przywilej: tylko pobieranie). [unit 5]

W laboratorium tworzysz aplikację od razu z uwierzytelnianiem, bo pierwsza rewizja musi móc pobrać obraz: flaga `--registry-identity system` razem z `--registry-server`; CLI **automatycznie przypisuje** rolę `AcrPull`, gdy używasz jej z ACR. [unit 7] Dokumentacja potwierdza automatyczne przypisanie roli w CLI (w PowerShell trzeba przypisać rolę ręcznie), a portal próbuje ją dodać sam. Gdy pobranie obrazu się nie udaje, sprawdź rolę `AcrPull`; role potrzebują minuty lub dwóch na propagację. [docs: https://learn.microsoft.com/azure/container-apps/managed-identity-image-pull] [unit 7]

## D4. Weryfikacja konfiguracji rejestru i dobre praktyki
_Źródło: [unit 5]_

```azurecli
az containerapp registry list -n ai-api -g rg-aca-demo

az containerapp registry show -n ai-api -g rg-aca-demo \
    --server myregistry.azurecr.io
```

Praktyki: preferuj tożsamość zarządzaną dla ACR (`--identity`); nadawaj najmniejsze uprawnienia (`AcrPull`); używaj w pełni kwalifikowanych nazw obrazów z nazwą hosta rejestru; po zmianie wdróż nową rewizję i sprawdź, czy startuje, zanim uznasz, że zmiana jest poprawna.

---

# CZĘŚĆ E - WERYFIKACJA WDROŻENIA

## E1. Konfiguracja, ingress i logi
_Źródło: [unit 6]_

Weryfikacja domyka pętlę między zamiarem a tym, co naprawdę działa. W usługach AI błędy startu łatwo wziąć za problem z modelem, brakujący sekret lub błąd rejestru. Dwa rodzaje logów odpowiadają na różne pytania: **logi konsoli** (stdout/stderr Twojego kontenera) i **logi systemowe** platformy.

```azurecli
az containerapp show -n ai-api -g rg-aca-demo                       # (1)
az containerapp logs show -n ai-api -g rg-aca-demo                  # (2)
az containerapp logs show -n ai-api -g rg-aca-demo --follow --tail 30   # (3)
az containerapp logs show -n ai-api -g rg-aca-demo --type system   # (4)
```
Objaśnienie: (1) konfiguracja i FQDN, widać np. `internal` zamiast `external`; (2) ostatnie linie logów konsoli; (3) strumień na żywo (`--follow`) z większą liczbą linii; (4) logi systemowe.

Sprawdzone w dokumentacji: `--tail` przyjmuje 0-300 i domyślnie wynosi **20**; `--type` to `console` (domyślnie) lub `system`; logi pochodzą tylko z **jednej** rewizji, repliki i kontenera (domyślnie z najnowszej rewizji). [docs: https://learn.microsoft.com/en-us/cli/azure/containerapp/logs]

## E2. Rewizje i repliki
_Źródło: [unit 6]_

```azurecli
az containerapp revision list -n ai-api -g rg-aca-demo
az containerapp revision list -n ai-api -g rg-aca-demo --all
az containerapp replica list -n ai-api -g rg-aca-demo
az containerapp replica list -n ai-api -g rg-aca-demo --revision MyRevision
```

`revision list` pokazuje, która rewizja jest aktywna, a które nieaktywne; `--all` dodaje nieaktywne do głębszego debugowania. **Replika** to działająca instancja rewizji; liczba i stan replik mówią, czy aplikacja działa i czy skaluje się jak trzeba (zimny start i skalowanie dodają opóźnień).

## E3. Dobre praktyki weryfikacji
_Źródło: [unit 6]_

- **Zacznij od logów** (`az containerapp logs show`): błędy startu i działania.
- **Rewizje do walidacji wdrożenia:** po aktualizacji sprawdź, czy powstała nowa rewizja i czy jest aktywna.
- **Repliki podczas incydentów:** wykryjesz scale-to-zero (skalowanie do zera), pętle awarii lub brak ruchu.
- **Wyjścia przyjazne automatyzacji:** `--query` wyciąga pola, które często sprawdzasz (FQDN, stan rewizji).

---

# CZĘŚĆ F - ĆWICZENIA

Przeczytaj wcześniej części A-E. Każde ćwiczenie ma ten sam układ pól (przewodnik stylu §5). Ćwiczenia I1 i I2 tworzą płatne zasoby w chmurze: rób je w osobnych grupach zasobów i zawsze wykonuj **Sprzątanie**. Kod aplikacji i SDK jest w Pythonie; odpowiedniki C# są w ćwiczeniach z kodem.

> Wersje sprawdzone 2026-11-22 (źródło: pypi.org, api.nuget.org, mcr.microsoft.com, `az version`): Python 3.12 (obraz `python:3.12-slim`, zgodny z wymaganiem modułu "Python 3.12 or greater" [unit 7]), Flask 3.1.3, gunicorn 27.0.1, azure-mgmt-appcontainers 5.1.0, azure-identity 1.27.0; C#: .NET SDK 10.0 (obrazy `mcr.microsoft.com/dotnet/sdk:10.0` i `aspnet:10.0` istnieją), Azure.ResourceManager.AppContainers 1.6.0, Azure.Identity 1.21.0; Azure CLI 2.90.0, rozszerzenie containerapp 1.3.0b6. Starter laboratorium Microsoftu pinuje starsze `flask==3.0.0` i `gunicorn==22.0.0`. Przed użyciem sprawdź PyPI i NuGet.

## I1. API w prywatnym ACR z tożsamością zarządzaną, sekretem i weryfikacją (lab modułu)
(rozwinięcie jednostki 7; źródło scenariusza: [unit 7])

**Gdzie / Plik:** terminal z Azure CLI (Bash) w pustym katalogu; pliki `api/main.py`, `api/requirements.txt`, `api/Dockerfile`.

Lab Microsoftu daje paczkę startową i skrypt `azdeploy.py`. Tutaj budujesz to samo ręcznie, żeby zobaczyć każdy krok. [unit 7]

> Uwaga o labie: lab wymaga planu Pay-As-You-Go lub innego płatnego, bo zadania ACR Tasks są tymczasowo wstrzymane dla darmowych kredytów Azure [unit 7].

`api/main.py` (pełny, sprawdzony lokalnie klientem testowym Flask):
```python
import os

from flask import Flask, jsonify

app = Flask(__name__)

MODEL_NAME = os.getenv("MODEL_NAME", "not-configured")
EMBEDDINGS_API_KEY = os.getenv("EMBEDDINGS_API_KEY", "")


@app.get("/health")
def health():
    return jsonify(status="healthy")


@app.get("/")
def root():
    # Nigdy nie zwracamy samego sekretu - tylko informacje, czy jest ustawiony.
    return jsonify(
        model=MODEL_NAME,
        embeddings_api_key_configured=bool(EMBEDDINGS_API_KEY.strip()),
        embeddings_api_key_length=len(EMBEDDINGS_API_KEY),
    )
```
`api/requirements.txt`:
```
flask==3.1.3
gunicorn==27.0.1
```
`api/Dockerfile`:
```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY main.py .
ENV PORT=8000
EXPOSE 8000
CMD ["sh", "-c", "gunicorn --bind 0.0.0.0:${PORT:-8000} --workers 2 --access-logfile - main:app"]
```
Polecenia (zmienne ustaw raz, w tej samej sesji):
```bash
RG=rg-aca-deploy-i1; LOC=westeurope
ACR_NAME=acrdeploy$RANDOM$RANDOM                    # (1)
az provider register --namespace Microsoft.App
az provider register --namespace Microsoft.OperationalInsights
az provider register --namespace Microsoft.ContainerRegistry
az extension add --name containerapp --upgrade

az group create --name $RG --location $LOC
az acr create --resource-group $RG --name $ACR_NAME --sku Basic --admin-enabled false   # (2)
az acr build --registry $ACR_NAME --image ai-api:v1 --file api/Dockerfile api           # (3)

az containerapp env create --name aca-env-i1 --resource-group $RG --location $LOC       # (4)

az containerapp create --name ai-api --resource-group $RG --environment aca-env-i1 \
    --image $ACR_NAME.azurecr.io/ai-api:v1 --ingress external --target-port 8000 \
    --env-vars MODEL_NAME=demo-model \
    --registry-server $ACR_NAME.azurecr.io --registry-identity system                    # (5)

az containerapp secret set -n ai-api -g $RG --secrets embeddings-api-key=demo-key-12345  # (6)
az containerapp update -n ai-api -g $RG \
    --set-env-vars EMBEDDINGS_API_KEY=secretref:embeddings-api-key                       # (7)
az containerapp revision list -n ai-api -g $RG -o table                                  # (8)

FQDN=$(az containerapp show -n ai-api -g $RG --query properties.configuration.ingress.fqdn -o tsv)
curl -s https://$FQDN/health                                                             # (9)
curl -s https://$FQDN/
az containerapp logs show -n ai-api -g $RG                                               # (10)
```
Objaśnienie: (1) nazwa rejestru musi być globalnie unikalna; (2) najtańszy SKU, bez konta administratora (hasła admina nie używamy); (3) `az acr build` buduje obraz w chmurze, bez lokalnego Dockera; (4) środowisko; (5) `--registry-identity system` + `--registry-server`: pierwsza rewizja może pobrać obraz; CLI przypisuje rolę `AcrPull` [unit 7]; (6) sekret; (7) odwołanie z zmiennej, to tworzy nową rewizję (zmienna jest revision-scope); (8) lista rewizji - nazwa nowej kończy się sufiksem w rodzaju `--0000002` [unit 7]; (9) sprawdzenie; (10) logi - szukaj komunikatów `gunicorn` o uruchomieniu procesów i nasłuchu na porcie 8000 oraz wpisów żądań. [unit 7]

Typowe problemy z labu [unit 7]: błędy `ImagePullBackOff` / `unauthorized` / 403 przy pobieraniu obrazu - sprawdź `az containerapp identity show` i rolę **AcrPull** (propagacja roli trwa minutę lub dwie); 502/503 tuż po wdrożeniu - odczekaj minutę, bo pierwszy start pobiera i uruchamia kontener; nieudany build - `az acr task list-runs --registry $ACR_NAME --output table` i `az acr task logs --registry $ACR_NAME --run-id <run-id>`; brak zmiennych w sesji - w labie wczytujesz je z pliku `.env` (`source .env`).

**Koszt:** ACR **Basic** jest naliczany stałą stawką dobową niezależnie od użycia (stawkę sprawdź na stronie cennika ACR: Niesprawdzone - zweryfikuj w https://azure.microsoft.com/pricing/details/container-registry/). Container Apps (plan Consumption): pierwsze 180 000 vCPU-sekund, 360 000 GiB-sekund i 2 mln żądań HTTP w miesiącu na subskrypcję są bezpłatne; replika skalowana do zera nie kosztuje; środowisko z własną siecią lub innymi zasobami może dodać opłaty. [docs: https://learn.microsoft.com/azure/container-apps/billing] Log Analytics tworzony z środowiskiem rozlicza ilość przyjętych logów (przy kilku żądaniach grosze). Kilka godzin pracy to rząd groszy do kilku złotych, o ile wykonasz Sprzątanie.

**Zadania:** (a) wykonaj polecenia i potwierdź, że `/health` i `/` odpowiadają; (b) **zepsuj to - rotacja sekretu:** zmień wartość sekretu na dłuższą (`az containerapp secret set -n ai-api -g $RG --secrets embeddings-api-key=a-much-longer-key-0987654321`), wywołaj `/`, potem zrestartuj rewizję (`az containerapp revision restart -n ai-api -g $RG --revision <aktywna-rewizja>`) i wywołaj `/` ponownie; (c) **zepsuj to - zły obraz:** `az containerapp update -n ai-api -g $RG --image $ACR_NAME.azurecr.io/ai-api:v999`, zobacz `revision list`, logi systemowe (`--type system`) i czy `/health` nadal odpowiada; (d) przywróć dobry obraz `:v1`.

**Sprawdź:** (a) `/health` zwraca `{"status":"healthy"}`, a `/` pokazuje `model` równy `demo-model`, `embeddings_api_key_configured` równe `true` i `embeddings_api_key_length` równe 14 (długość `demo-key-12345`); (b) po samej zmianie sekretu długość nadal wynosi 14 (nowa rewizja nie powstała, replika ma starą wartość), po restarcie rewizji długość wynosi 28 [docs: https://learn.microsoft.com/azure/container-apps/manage-secrets]; (c) nowa rewizja nie staje się zdrowa, a logi systemowe pokazują problem z pobraniem obrazu; dokładny komunikat zależy od platformy, a to, czy poprzednia rewizja nadal obsługuje ruch w trybie Single, sprawdź sam i zapisz obserwację [autor: niezweryfikowane]; (d) po przywróceniu `:v1` powstaje kolejna rewizja i aplikacja działa.

**Sprzątanie:**
```bash
az group delete --name $RG --yes --no-wait
az group exists --name $RG      # po kilku minutach powinno wypisać false
```
Usuwana jest cała grupa (ACR, środowisko, Log Analytics, aplikacja). Uważaj, by nie użyć istniejącej grupy z innymi zasobami.

## I2. Wdrożenie plikiem YAML, internal vs external i `up`
(rozwinięcie sekcji B2, B5, A3)

**Gdzie / Plik:** terminal z Azure CLI (Bash); plik `containerapp.yml`.

Wymaga środowiska. Ćwiczenie tworzy własne, osobne (publiczny obraz, bez rejestru):
```bash
RG=rg-aca-deploy-i2; LOC=westeurope
az group create --name $RG --location $LOC
az containerapp env create --name aca-env-i2 --resource-group $RG --location $LOC
ENV_ID=$(az containerapp env show --name aca-env-i2 --resource-group $RG --query id -o tsv)
```
`containerapp.yml` (podstaw `<ID środowiska>` wartością `$ENV_ID`):
```yaml
location: westeurope
name: hello-yaml
type: Microsoft.App/containerApps
properties:
  managedEnvironmentId: <ID środowiska>
  configuration:
    activeRevisionsMode: Single
    ingress:
      external: true
      targetPort: 80
  template:
    containers:
    - name: hello
      image: mcr.microsoft.com/k8se/quickstart:latest
      env:
      - name: LOG_LEVEL
        value: info
    scale:
      minReplicas: 0
      maxReplicas: 1
```
```bash
az containerapp create --name hello-yaml --resource-group $RG --yaml ./containerapp.yml   # (1)
az containerapp show -n hello-yaml -g $RG --query properties.configuration.ingress.external   # (2)
az containerapp ingress update -n hello-yaml -g $RG --type internal                       # (3)
az containerapp revision list -n hello-yaml -g $RG -o table                               # (4)
```
Objaśnienie: (1) `--yaml` ignoruje inne flagi; (2) odczyt ustawienia ingress; (3) zmiana ingress na wewnętrzny; (4) liczba rewizji przed i po zmianie.

**Koszt:** środowisko i aplikacja z `minReplicas: 0` w planie Consumption nie naliczają kosztu przy zerowym ruchu (darmowe progi miesięczne jak w I1) [docs: https://learn.microsoft.com/azure/container-apps/billing]; Log Analytics rozlicza ilość logów (pomijalna).

**Zadania:** (a) wdróż YAML i otwórz `https://<fqdn>` (FQDN z `az containerapp show -n hello-yaml -g $RG --query properties.configuration.ingress.fqdn -o tsv`); (b) zmień zmienną `LOG_LEVEL` w YAML na `debug`, wykonaj `az containerapp update --yaml ./containerapp.yml` i policz rewizje; (c) **zepsuj to:** zmień ingress na `internal` poleceniem (3), sprawdź, czy rewizji przybyło, i wywołaj stary adres z zewnątrz; (d) uruchom `az containerapp up` z inną nazwą aplikacji w tym samym środowisku i sprawdź, że środowisko nie zostało utworzone drugi raz.

**Sprawdź:** (a) strona przykładowa quickstart jest dostępna; (b) powstaje nowa rewizja (zmienna to zmiana revision-scope) [docs: https://learn.microsoft.com/azure/container-apps/revisions]; (c) zmiana ingress **nie tworzy** nowej rewizji (application-scope), a żądanie spoza środowiska kończy się odrzuceniem (404 z serwera pośredniczącego środowiska, wg dokumentacji dla ingress `internal`) [docs: https://learn.microsoft.com/azure/container-apps/ingress-overview]; (d) `up` użył istniejącego środowiska i grupy [docs: https://learn.microsoft.com/azure/container-apps/get-started].

**Sprzątanie:**
```bash
az group delete --name $RG --yes --no-wait
az group exists --name $RG
```

## I3. Rewizje i repliki przez SDK: Python i odpowiednik C# (.NET)
(rozwinięcie sekcji E2; wymaga działającej aplikacji `ai-api` z I1, bez Sprzątania I1)

**Gdzie / Plik:** `list_revisions.py` (Python) oraz `Program.cs` (C#); zalogowany `az login`.

```bash
pip install azure-mgmt-appcontainers==5.1.0 azure-identity==1.27.0
export AZURE_SUBSCRIPTION_ID=$(az account show --query id -o tsv)
export RESOURCE_GROUP=rg-aca-deploy-i1 CONTAINER_APP_NAME=ai-api
```
`list_revisions.py`:
```python
import os

from azure.identity import DefaultAzureCredential
from azure.mgmt.appcontainers import ContainerAppsAPIClient

subscription_id = os.environ["AZURE_SUBSCRIPTION_ID"]
resource_group = os.environ["RESOURCE_GROUP"]
app_name = os.environ["CONTAINER_APP_NAME"]

client = ContainerAppsAPIClient(DefaultAzureCredential(), subscription_id)  # (1)

for rev in client.container_apps_revisions.list_revisions(resource_group, app_name):  # (2)
    p = rev.properties
    print(rev.name, "active =", p.active, "traffic =", p.traffic_weight,
          "health =", p.health_state, "provisioning =", p.provisioning_state)  # (3)
    replicas = client.container_apps_revision_replicas.list_replicas(
        resource_group, app_name, rev.name)  # (4)
    for replica in replicas.value:  # (5)
        print("   replica", replica.name, replica.properties.running_state)
```
Objaśnienie: (1) klient zarządzania `ContainerAppsAPIClient` z `DefaultAzureCredential` (łańcuch źródeł poświadczeń: zmienne środowiskowe, `az login` itd.), wzorzec z dokumentacji [docs: https://learn.microsoft.com/python/api/overview/azure/mgmt-appcontainers-readme?view=azure-python]; (2) lista rewizji; (3) pola `active`, `traffic_weight`, `health_state`, `provisioning_state` istnieją w `RevisionProperties` (sprawdzone w pakiecie, 2026-11-22) [autor: test 2026-11-22]; (4)-(5) lista replik rewizji zwraca kolekcję z polem `value`.

Skrypt skompilowano (`py_compile`) i sprawdzono nazwy modeli w pakiecie; nie uruchamiano go na prawdziwej subskrypcji [autor: test 2026-11-22].

**Odpowiednik C# (.NET)**

Różnice: zamiast jednego klienta z operacjami (`container_apps_revisions`) pracujesz na **obiektach zasobów** ARM (`ContainerAppResource`, potem kolekcje `GetContainerAppRevisions()`), a listy są asynchroniczne (`await foreach`). Dane zasobu masz w `.Data`; właściwość `active` z Pythona to `IsActive`, `traffic_weight` to `TrafficWeight`. Nowy projekt: `dotnet new console -f net10.0`, potem `dotnet add package Azure.ResourceManager.AppContainers --version 1.6.0` i `dotnet add package Azure.Identity --version 1.21.0`.
```csharp
using Azure.Identity;
using Azure.ResourceManager;
using Azure.ResourceManager.AppContainers;

string sub = Environment.GetEnvironmentVariable("AZURE_SUBSCRIPTION_ID")!;
string rg = Environment.GetEnvironmentVariable("RESOURCE_GROUP")!;
string appName = Environment.GetEnvironmentVariable("CONTAINER_APP_NAME")!;

var arm = new ArmClient(new DefaultAzureCredential());
ContainerAppResource app = arm.GetContainerAppResource(
    ContainerAppResource.CreateResourceIdentifier(sub, rg, appName));

await foreach (ContainerAppRevisionResource rev in app.GetContainerAppRevisions().GetAllAsync())
{
    var d = rev.Data;
    Console.WriteLine($"{d.Name} active={d.IsActive} traffic={d.TrafficWeight} " +
                      $"health={d.HealthState} state={d.ProvisioningState}");
    await foreach (ContainerAppReplicaResource rep in rev.GetContainerAppReplicas().GetAllAsync())
        Console.WriteLine($"  replica {rep.Data.Name} {rep.Data.RunningState}");
}
```
[docs: https://learn.microsoft.com/dotnet/api/overview/azure/resourcemanager.appcontainers-readme?view=azure-dotnet] [autor: test 2026-11-22] (kod skompilowano na .NET SDK 10.0.401 z Azure.ResourceManager.AppContainers 1.6.0; nie uruchamiano go wobec prawdziwej subskrypcji).

Aplikacja z I1 w C#: ten sam kontrakt (`/health`, `/` ze zmiennymi `MODEL_NAME` i `EMBEDDINGS_API_KEY`) daje minimalne API ASP.NET Core (`dotnet new web -f net10.0`), sprawdzone kompilacją: [autor: test 2026-11-22]
```csharp
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

string modelName = Environment.GetEnvironmentVariable("MODEL_NAME") ?? "not-configured";
string apiKey = Environment.GetEnvironmentVariable("EMBEDDINGS_API_KEY") ?? "";

app.MapGet("/health", () => Results.Json(new { status = "healthy" }));

// Nigdy nie zwracamy samego sekretu - tylko informacje, czy jest ustawiony.
app.MapGet("/", () => Results.Json(new
{
    model = modelName,
    embeddings_api_key_configured = !string.IsNullOrWhiteSpace(apiKey),
    embeddings_api_key_length = apiKey.Length
}));

app.Run();
```
Różnice kontenerowe: obrazy `mcr.microsoft.com/dotnet/sdk:10.0` (build) i `mcr.microsoft.com/dotnet/aspnet:10.0` (uruchomienie); od .NET 8 obrazy ASP.NET Core nasłuchują domyślnie na porcie **8080**, nie 80, więc daj `--target-port 8080` (albo ustaw `ASPNETCORE_HTTP_PORTS`). [docs: https://learn.microsoft.com/en-us/dotnet/core/compatibility/containers/8.0/aspnet-port]

**Koszt:** brak kosztu własnego (odczyt metadanych); koszt aplikacji z I1 jak tam.

**Zadania:** (a) uruchom `python list_revisions.py` i porównaj wynik z `az containerapp revision list -o table`; (b) **zepsuj to:** ustaw `CONTAINER_APP_NAME=nie-ma-takiej` i przeczytaj błąd; (c) uruchom wersję C# (`dotnet run`) i porównaj kolumny.

**Sprawdź:** (a) te same rewizje, nazwy i stan aktywności; (b) błąd typu "nie znaleziono zasobu" (ResourceNotFound) z nazwą aplikacji; dokładny tekst zależy od wersji SDK [autor: niezweryfikowane]; (c) ten sam zestaw rewizji w formacie z `IsActive` i `TrafficWeight`.

**Sprzątanie:** brak (skrypty tylko czytają); po I3 wykonaj Sprzątanie I1.

---

# CZĘŚĆ G - QUIZ Z MODUŁU MICROSOFTU (5 pytań) + wyjaśnienia

Learn nie pokazuje poprawnych odpowiedzi w ocenie modułu, więc odpowiedzi są wnioskowane z treści jednostek; przy każdym pytaniu jest krótki znacznik.

## G1. Który zasób daje wspólną granicę sieci i logów dla wielu aplikacji?
-> *Container Apps environment.* Rewizja to wersja konfiguracji jednej aplikacji, replika to działająca instancja - żadne z nich nie jest wspólną granicą. *(odpowiedź wnioskowana: unit 2, sekcja "What a Container Apps environment provides")*

## G2. Spójne wdrożenie z kontrolą wersji i review konfiguracji
-> *`az containerapp create --yaml` i `az containerapp update --yaml` z plikiem YAML w repozytorium.* Same `--set-env-vars` bez pliku nie dają jednego recenzowalnego źródła prawdy; sama zmiana tagu obrazu nie przenosi konfiguracji środowiskowej. *(odpowiedź wnioskowana: unit 3, sekcja "Use YAML for consistent, reviewable deployments")*

## G3. Klucz API dostawcy embeddingów bez trzymania go w YAML
-> *Sekret Container Apps i odwołanie z zmiennej środowiskowej (`secretref:`).* Jawny tekst w `env:` i wypiekanie klucza w obrazie ujawniają wartość. *(odpowiedź wnioskowana: unit 4, sekcja "Store sensitive settings as Container Apps secrets")*

## G4. Które polecenie pokazuje wersjonowaną zmianę po aktualizacji?
-> *`az containerapp revision list`.* `registry list` pokazuje rejestry, `secret list` sekrety. *(odpowiedź wnioskowana: unit 6, sekcja "Check revisions to confirm which version is active")*

## G5. Aplikacja nie startuje po zmianie obrazu - pierwszy krok diagnozy
-> *`az containerapp logs show`.* Logi dają najszybszy wgląd w błąd startu; `revision list` i `replica list` pokazują stan, ale nie przyczynę. *(odpowiedź wnioskowana: unit 6, sekcja "Best practices for verification: Start with logs")*

---

# CZĘŚĆ H - PYTANIA KONTROLNE (odpowiedz bez zaglądania)

## H1. Co współdzielą aplikacje w jednym środowisku Container Apps?
<details><summary>Odpowiedź</summary>

Sieć wirtualną i miejsce zapisu logów (domyślnie wspólny obszar Log Analytics). Środowiska rozdziela się według cyklu życia (dev/test/prod).
</details>

## H2. Czym różni się `az containerapp up` od `az containerapp create`?
<details><summary>Odpowiedź</summary>

`up` jest szybkie i potrafi utworzyć środowisko oraz zasoby pomocnicze (a istniejące zasoby wykorzysta); `create` jest jawne, gdy środowisko już istnieje i chcesz pełnej kontroli.
</details>

## H3. Które zmiany tworzą nową rewizję, a które nie?
<details><summary>Odpowiedź</summary>

Nową rewizję tworzą zmiany revision-scope (sekcja `template`: obraz, zmienne, reguły skalowania, sufiks). Zmiany application-scope (sekcja `configuration`: wartości sekretów, ingress, tryb rewizji, poświadczenia rejestru, Dapr) nie tworzą rewizji.
</details>

## H4. Jak przekazać sekret do kontenera jako zmienną środowiskową?
<details><summary>Odpowiedź</summary>

Utwórz sekret (`az containerapp secret set --secrets nazwa=wartość`), potem ustaw zmienną `ZMIENNA=secretref:nazwa` (w YAML: pole `secretRef`). Po zmianie wartości sekretu trzeba wdrożyć nową rewizję lub zrestartować istniejącą.
</details>

## H5. Jaka rola i jaka flaga pozwalają ACA pobierać obrazy z ACR bez hasła?
<details><summary>Odpowiedź</summary>

Rola `AcrPull` dla tożsamości aplikacji na rejestrze; flaga `--identity system` w `az containerapp registry set` (lub `--registry-identity system` z `--registry-server` przy `create`).
</details>

## H6. Co robi `--set-env-vars`, a co `--replace-env-vars`?
<details><summary>Odpowiedź</summary>

`--set-env-vars` dodaje lub zmienia wskazane zmienne i nie rusza pozostałych; `--replace-env-vars` zastępuje cały zestaw (pozostałe zmienne znikają).
</details>

## H7. Jakie są domyślne wartości `az containerapp logs show` i jak zobaczyć logi platformy?
<details><summary>Odpowiedź</summary>

Domyślnie typ `console` i 20 linii (`--tail` przyjmuje 0-300). Logi platformy: `--type system`; strumień na żywo: `--follow`.
</details>

## H8. Obraz `MyRegistry.azurecr.io/Ai-Api:v1` nie chce się pobrać. Co sprawdzisz najpierw?
<details><summary>Odpowiedź</summary>

Wielkie litery w ścieżce obrazu (ścieżka musi być małymi literami; błąd wygląda jak problem z uwierzytelnianiem), potem uwierzytelnianie do rejestru i rolę `AcrPull`.
</details>

---

# CZĘŚĆ J - CO DALEJ

## J1. Następny krok i pokrycie egzaminu
_Źródło: [moduł kursu: manage-containers-azure-container-apps] + [moduł kursu: scale-containers-azure-container-apps]_

Dalej: moduł 05 (zarządzanie: aktualizacje obrazów, rewizje, cykl życia, health probes, optymalizacja ustawień kontenera), moduł 06 (skalowanie, KEDA, tryby rewizji i ruch), moduł 07 (dynamic sessions). Wróć do części B i C, gdy będziesz rozwiązywać zadania o rewizjach i sekretach.

**Pokrycie egzaminu** (wiersze z `sources/_exam/skill-map.md`, stan study guide z 2026-10-30):

| Umiejętność (study guide) | Domena | Status | Gdzie |
|---|---|---|---|
| Deploy applications to Azure Container Apps, including environment configuration and revision management | 1 | uczone tutaj (wdrożenie, środowisko, rewizje); zarządzanie rewizjami także w module 05 | `04-deploy-containers-azure-container-apps.md` §A1-§E3; `05-manage-containers-azure-container-apps.md` |
| Implement event-driven scaling by using KEDA in Container Apps | 1 | uczone w module 06 | https://learn.microsoft.com/en-us/training/modules/scale-containers-azure-container-apps/ |
| Monitor and troubleshoot solutions on AKS and Container Apps by inspecting logs, events, and end-to-end connectivity | 1 | tu tylko wzmianka (logi i rewizje w §E1-§E2); właściciel Container Apps: moduł 05 | `05-manage-containers-azure-container-apps.md` |
| Build, store, version, and manage container images by using Azure Container Registry | 1 | uczone w module 01 (tu tylko pobieranie obrazu z ACR, §D3) | `01-store-manage-containers-azure-container-registry.md` |

## J2. Uczciwie o egzaminie
_Źródło: [autor]_

Ten moduł pokrywa wdrożenie (CLI, YAML), konfigurację środowiska, zmienne i sekrety, pobieranie obrazów z prywatnego ACR oraz podstawową weryfikację. Nie uczy: skalowania i KEDA (moduł 06), health probes, cyklu życia i aktualizacji obrazów w trybie wielu rewizji (moduł 05), dynamic sessions (moduł 07). Quiz modułu jest krótki (5 pytań), więc egzamin będzie głębszy: ćwicz rozróżnianie revision-scope i application-scope, `secretref:` oraz `--identity system`/`AcrPull`.
