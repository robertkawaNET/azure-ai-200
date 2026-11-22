# Moduł 02: Kontenery w Azure App Service (źródła obrazu, runtime, ustawienia aplikacji, diagnostyka)

Moduł Microsoft Learn: **Deploy containers to Azure App Service** - https://learn.microsoft.com/en-us/training/modules/deploy-containers-azure-app-service/
Ścieżka: Implement container application hosting on Azure (moduł 2 z 3), kurs AI-200T00, egzamin **AI-200** (Azure AI Cloud Developer Associate), domena 1 "Develop containerized solutions on Azure".
Cel: po tym module umiesz wdrożyć własny kontener z rejestru do Azure App Service, ustawić port, polecenie startowe, trwały magazyn i kontrolę stanu, przekazać konfigurację przez ustawienia aplikacji (w tym sekrety z Key Vault) oraz diagnozować problemy logami i konsolą Kudu.
Czas: ok. 5-7 h (czytanie + trzy ćwiczenia). Ćwiczenia robisz w Azure Cloud Shell (Bash) albo lokalnie z Azure CLI 2.x. Potrzebujesz subskrypcji **płatnej**: ćwiczenie buduje obraz przez ACR Tasks, a uruchomienia ACR Tasks są tymczasowo wstrzymane dla darmowych kredytów Azure. Koszt: rejestr ACR Basic i plan App Service B1 (płatne); szczegóły i sprzątanie w części E.

Język kodu: egzamin AI-200 zakłada programowanie w **Pythonie**, więc kod aplikacji jest w Pythonie. W ćwiczeniu I1 dodano blok **Odpowiednik C# (.NET)** z opisem różnic (m.in. inny domyślny port). Polecenia `az`, Dockerfile i teoria są wspólne dla obu języków.

Znaczniki źródeł (linia `_Źródło: ..._` pod nagłówkiem):
- `[unit N]` - jednostka (unit) modułu Microsoft Learn; N to prefiks pliku w `sources/deploy-containers-azure-app-service/` (`6-lab` = README laboratorium, `6-starter` = pliki startowe laboratorium).
- `[moduł kursu: <slug>]` - inny moduł tego samego kursu.
- `[docs: URL]` - oficjalna dokumentacja produktu.
- `[autor]` - dodane przez autora tutoriala; `[autor: test <data>]` - sprawdzone testem w danym dniu; `[autor: niezweryfikowane]` - jeszcze nie sprawdzone w Azure.

W tekście "moduł N" znaczy plik tutoriala `0N-*.md` (moduł 01 = `01-store-manage-containers-azure-container-registry.md`).

---

<!-- toc -->
**Spis treści**

- **[CZĘŚĆ A - WDRAŻANIE KONTENERA DO APP SERVICE](#część-a---wdrażanie-kontenera-do-app-service)**
  - [A1. Czym jest App Service dla kontenerów i po co](#a1-czym-jest-app-service-dla-kontenerów-i-po-co)
  - [A2. Źródła obrazu](#a2-źródła-obrazu)
  - [A3. Wdrożenie przez portal Azure](#a3-wdrożenie-przez-portal-azure)
  - [A4. Uwierzytelnianie do ACR: tożsamość zarządzana albo dane administratora](#a4-uwierzytelnianie-do-acr-tożsamość-zarządzana-albo-dane-administratora)
  - [A5. Wdrożenie z CLI, z innych rejestrów i z VS Code](#a5-wdrożenie-z-cli-z-innych-rejestrów-i-z-vs-code)
  - [A6. Aktualizacja obrazu, ciągłe wdrażanie, pobieranie warstw i weryfikacja](#a6-aktualizacja-obrazu-ciągłe-wdrażanie-pobieranie-warstw-i-weryfikacja)
- **[CZĘŚĆ B - ZACHOWANIE KONTENERA W RUNTIME](#część-b---zachowanie-kontenera-w-runtime)**
  - [B1. Polecenie startowe](#b1-polecenie-startowe)
  - [B2. Port kontenera (`WEBSITES_PORT`)](#b2-port-kontenera-websites_port)
  - [B3. Trwały magazyn (`WEBSITES_ENABLE_APP_SERVICE_STORAGE`)](#b3-trwały-magazyn-websites_enable_app_service_storage)
  - [B4. Always On](#b4-always-on)
  - [B5. Kontrola stanu (health check)](#b5-kontrola-stanu-health-check)
- **[CZĘŚĆ C - USTAWIENIA APLIKACJI I SEKRETY](#część-c---ustawienia-aplikacji-i-sekrety)**
  - [C1. Ustawienia aplikacji (app settings)](#c1-ustawienia-aplikacji-app-settings)
  - [C2. Connection strings](#c2-connection-strings)
  - [C3. Edycja zbiorcza](#c3-edycja-zbiorcza)
  - [C4. Ustawienia slotów (slot settings)](#c4-ustawienia-slotów-slot-settings)
  - [C5. Odwołania do Key Vault](#c5-odwołania-do-key-vault)
  - [C6. Weryfikacja konfiguracji](#c6-weryfikacja-konfiguracji)
- **[CZĘŚĆ D - OBSERWOWANIE I DIAGNOZOWANIE KONTENERA](#część-d---obserwowanie-i-diagnozowanie-kontenera)**
  - [D1. Logi kontenera](#d1-logi-kontenera)
  - [D2. Strumień logów (log stream)](#d2-strumień-logów-log-stream)
  - [D3. Konsola diagnostyczna Kudu (SCM)](#d3-konsola-diagnostyczna-kudu-scm)
  - [D4. Diagnostyka platformowa i Log Analytics](#d4-diagnostyka-platformowa-i-log-analytics)
  - [D5. Dostęp SSH do kontenera](#d5-dostęp-ssh-do-kontenera)
  - [D6. Typowe problemy i rozwiązania](#d6-typowe-problemy-i-rozwiązania)
- **[CZĘŚĆ E - ĆWICZENIA](#część-e---ćwiczenia)**
  - [I1. Wdrożenie kontenera z ACR do App Service z tożsamością zarządzaną - Python i C#](#i1-wdrożenie-kontenera-z-acr-do-app-service-z-tożsamością-zarządzaną---python-i-c)
  - [I2. Ustawienia aplikacji, trwały magazyn, Always On, health check i slot settings](#i2-ustawienia-aplikacji-trwały-magazyn-always-on-health-check-i-slot-settings)
  - [I3. Logi, strumień logów, Kudu i diagnoza błędnego portu](#i3-logi-strumień-logów-kudu-i-diagnoza-błędnego-portu)
- **[CZĘŚĆ F - QUIZ Z MODUŁU MICROSOFTU (5 pytań) + wyjaśnienia](#część-f---quiz-z-modułu-microsoftu-5-pytań--wyjaśnienia)**
  - [F1. Obraz nasłuchuje na porcie 8000, po wdrożeniu żądania kończą się błędami połączenia](#f1-obraz-nasłuchuje-na-porcie-8000-po-wdrożeniu-żądania-kończą-się-błędami-połączenia)
  - [F2. Pliki wyjściowe znikają po restarcie kontenera](#f2-pliki-wyjściowe-znikają-po-restarcie-kontenera)
  - [F3. Inny adres API dla staging i produkcji, który nie ma się zamienić podczas swap](#f3-inny-adres-api-dla-staging-i-produkcji-który-nie-ma-się-zamienić-podczas-swap)
  - [F4. Kontener startuje, ale health check zawodzi, a aplikacja ma endpoint `/healthz` zwracający 200](#f4-kontener-startuje-ale-health-check-zawodzi-a-aplikacja-ma-endpoint-healthz-zwracający-200)
  - [F5. Sprawdzenie, czy ustawienia aplikacji trafiły do działającego kontenera](#f5-sprawdzenie-czy-ustawienia-aplikacji-trafiły-do-działającego-kontenera)
- **[CZĘŚĆ G - PYTANIA KONTROLNE (odpowiedz bez zaglądania)](#część-g---pytania-kontrolne-odpowiedz-bez-zaglądania)**
  - [G1. Jakie dwie metody uwierzytelniania do ACR obsługuje App Service i którą zalecasz na produkcji?](#g1-jakie-dwie-metody-uwierzytelniania-do-acr-obsługuje-app-service-i-którą-zalecasz-na-produkcji)
  - [G2. Co się stanie, gdy wypchniesz nowy obraz pod ten sam tag, który App Service już wdrożył?](#g2-co-się-stanie-gdy-wypchniesz-nowy-obraz-pod-ten-sam-tag-który-app-service-już-wdrożył)
  - [G3. Do czego służy `WEBSITES_PORT` i jaki jest port domyślny?](#g3-do-czego-służy-websites_port-i-jaki-jest-port-domyślny)
  - [G4. Co robi `WEBSITES_ENABLE_APP_SERVICE_STORAGE` i jaki jest domyślny stan na Linuksie?](#g4-co-robi-websites_enable_app_service_storage-i-jaki-jest-domyślny-stan-na-linuksie)
  - [G5. Jaki poziom cenowy jest potrzebny dla Always On i co to robi?](#g5-jaki-poziom-cenowy-jest-potrzebny-dla-always-on-i-co-to-robi)
  - [G6. Jak działa health check w App Service?](#g6-jak-działa-health-check-w-app-service)
  - [G7. Czym różnią się app settings od connection strings i które wybierzesz dla Pythona?](#g7-czym-różnią-się-app-settings-od-connection-strings-i-które-wybierzesz-dla-pythona)
  - [G8. Jak zapewnić, że ustawienie nie zamieni się między slotami podczas swap, i jak odwołać się do sekretu z Key Vault?](#g8-jak-zapewnić-że-ustawienie-nie-zamieni-się-między-slotami-podczas-swap-i-jak-odwołać-się-do-sekretu-z-key-vault)
  - [G9. Wymień narzędzia diagnostyczne App Service i do czego służy każde.](#g9-wymień-narzędzia-diagnostyczne-app-service-i-do-czego-służy-każde)
- **[CZĘŚĆ H - CO DALEJ](#część-h---co-dalej)**
  - [H1. Podsumowanie, następny krok i pokrycie egzaminu](#h1-podsumowanie-następny-krok-i-pokrycie-egzaminu)
  - [H2. Uczciwie o egzaminie](#h2-uczciwie-o-egzaminie)

<!-- /toc -->

# CZĘŚĆ A - WDRAŻANIE KONTENERA DO APP SERVICE

## A1. Czym jest App Service dla kontenerów i po co
_Źródło: [unit 1] + [unit 2]_

**Azure App Service** to w pełni zarządzana platforma do hostowania aplikacji webowych. Możesz na niej uruchamiać **własny kontener** (Web App for Containers): przynosisz obraz z rejestru, a platforma zajmuje się przydzielaniem maszyn, równoważeniem obciążenia (load balancing) i skalowaniem. Dostajesz też sloty wdrożeń (deployment slots = osobne środowiska tej samej aplikacji, np. staging) i zintegrowaną diagnostykę. App Service uruchamia kontenery **Linux** (ten moduł dotyczy Linuksa).

Scenariusz z modułu: serwis przetwarzania dokumentów (przyjmuje dokumenty, wyciąga tekst i metadane, zwraca ustrukturyzowany wynik) w kontenerze. Wymagania: inna konfiguracja w każdym środowisku (dev: lokalny magazyn i szczegółowe logi; produkcja: konto Azure Storage i klucze API) **bez przebudowy obrazu**; widoczność działania (czy błąd to start kontenera, zła zmienna środowiskowa czy błąd aplikacji); automatyczne skalowanie przy zmiennym obciążeniu i krótki zimny start (cold start = start po bezczynności lub nowej instancji). [unit 1]

Cele modułu: wdrażać własne kontenery z rejestrów; konfigurować zachowanie runtime (polecenie startowe, port, trwały magazyn); konfigurować ustawienia aplikacji i connection strings; obserwować i diagnozować kontenery narzędziami diagnostycznymi App Service. [unit 1]

## A2. Źródła obrazu
_Źródło: [unit 2]_

Przy tworzeniu Web App for Containers wybierasz skąd App Service pobierze obraz (portal oferuje dwie opcje):

| Źródło | Kiedy | Szczegóły |
|---|---|---|
| **Azure Container Registry (ACR)** | zalecane dla produkcji | uwierzytelnianie przez Microsoft Entra ID, tożsamość zarządzana, geo-replikacja, skanowanie obrazów, dostęp prywatny; wybierasz rejestr, metodę uwierzytelniania, obraz i tag |
| **Inne rejestry** | Docker Hub, GitHub Container Registry, rejestry własne | dowolny rejestr dostępny przez HTTPS zgodny ze specyfikacją Docker Registry HTTP API V2; dla obrazów prywatnych podajesz adres serwera, użytkownika i hasło; dla publicznych tylko nazwę obrazu |

## A3. Wdrożenie przez portal Azure
_Źródło: [unit 2]_

Utworzenie aplikacji:

1. W portalu: **Create a resource** -> wyszukaj **Web App** -> **Create** -> **Web App**.
2. Karta **Basics**: subskrypcja, grupa zasobów, nazwa aplikacji, region.
3. **Publish**: **Container**; **Operating System**: **Linux**.
4. Wybierz plan App Service (App Service plan = zestaw maszyn i rozmiar, na których działa aplikacja) albo utwórz nowy.
5. Karta **Container**: konfiguracja źródła obrazu.

Dla ACR: **Image Source** = **Azure Container Registry**; wybierz **Registry** (rejestry z tej subskrypcji pojawiają się same); wybierz **Authentication** (**Managed Identity** albo **Admin credentials**); wybierz **Image** i **Tag**. Dla innych rejestrów: **Image Source** = **Other container registries**; dla prywatnych podaj **Server URL** (np. `https://index.docker.io/v1/` dla Docker Hub), **Username**, **Password**; podaj **Full Image Name and Tag** (np. `nginx:latest`, `myuser/myapp:v1`).

## A4. Uwierzytelnianie do ACR: tożsamość zarządzana albo dane administratora
_Źródło: [unit 2] + [unit 6-lab] + [docs: https://learn.microsoft.com/en-us/azure/app-service/configure-custom-container]_

- **Tożsamość zarządzana (managed identity)** - zalecana na produkcji. App Service uwierzytelnia się w ACR tożsamością Azure, a nie zapisanym hasłem: nie ma rotacji haseł i jest lepszy audyt. Tożsamość musi mieć w rejestrze rolę **AcrPull** (pobieranie obrazów; to zasada najmniejszych uprawnień: aplikacja może pobierać, ale nie wypychać ani administrować). Dwa rodzaje:
  - **system-assigned** - związana z cyklem życia aplikacji (tworzona po włączeniu, usuwana z aplikacją); prostsza, gdy tylko ta aplikacja jej potrzebuje;
  - **user-assigned** - osobny zasób Azure, który przypisujesz do jednej lub wielu aplikacji; wygodna, gdy wiele aplikacji ma ten sam dostęp lub chcesz nadać uprawnienia przed utworzeniem aplikacji.
- **Dane administratora (admin credentials)** - nazwa i hasło przechowywane w ACR. Prostsze w development (bez przypisywania ról), ale hasło trafia do konfiguracji App Service i trzeba je ręcznie rotować, gdy wycieknie. Wymaga włączenia konta admin w rejestrze:

```azurecli
az acr update --name myregistry --admin-enabled true
```

Przypisanie roli po stronie CLI (z laboratorium Microsoftu): `az webapp identity assign`, potem `az role assignment create --assignee <principalId> --scope <ID rejestru> --role AcrPull` i `az webapp config set --acr-use-identity true --acr-identity [system]`. Propagacja roli trwa minutę lub dwie. Laboratorium tworzy rejestr z wyłączonym kontem admin, żeby aplikacja nie polegała na wspólnych poświadczeniach. [unit 6-lab] Podczas ustawiania kontenera może pojawić się komunikat o braku poświadczeń do ACR; przy tożsamości zarządzanej można go zignorować (nie włączaj konta admin). [unit 6-lab]

## A5. Wdrożenie z CLI, z innych rejestrów i z VS Code
_Źródło: [unit 2]_

CLI daje powtarzalny przepływ do skryptu lub CI. Zakłada, że obraz jest już w ACR, a App Service ma uprawnienie do jego pobrania (zwykle tożsamość zarządzana z rolą AcrPull). Podaj pełny adres obrazu **z tagiem**, żeby wdrożenie było deterministyczne (patrz moduł 01, C1 i C4).

```azurecli
az webapp create \
    --resource-group myResourceGroup \
    --plan myAppServicePlan \
    --name myDocumentProcessor \
    --container-image-name myregistry.azurecr.io/docprocessor:v1
```

Inne rejestry:

```azurecli
# publiczny obraz z Docker Hub
az webapp create --resource-group myResourceGroup --plan myAppServicePlan --name myWebApp \
    --container-image-name nginx --docker-registry-server-url https://index.docker.io/v1/

# prywatny obraz
az webapp create --resource-group myResourceGroup --plan myAppServicePlan --name myWebApp \
    --container-image-name myusername/myapp:latest \
    --docker-registry-server-url https://index.docker.io/v1/ \
    --docker-registry-server-user myusername \
    --docker-registry-server-password <password>
```

Dla GitHub Container Registry użyj adresu serwera `https://ghcr.io`. **VS Code** z rozszerzeniami Docker i Azure App Service: zbuduj obraz lokalnie, wypchnij go do rejestru, w widoku REGISTRIES kliknij tag prawym przyciskiem -> **Deploy Image to Azure App Service** i przejdź kreator. Na produkcji sprawdź osobno uwierzytelnianie i role, żeby aplikacja mogła pobierać obraz.

## A6. Aktualizacja obrazu, ciągłe wdrażanie, pobieranie warstw i weryfikacja
_Źródło: [unit 2] + [docs: https://learn.microsoft.com/en-us/azure/app-service/configure-custom-container]_

Nowa wersja = nowy tag. Zmień skonfigurowany tag, a App Service sam zrestartuje aplikację i pobierze obraz na każdej instancji:

```azurecli
az webapp config container set \
    --resource-group myResourceGroup \
    --name myDocumentProcessor \
    --container-image-name myregistry.azurecr.io/docprocessor:v2
```

Jeśli wypchniesz nowy obraz pod **ten sam tag** (np. `latest`), App Service tego **nie wykryje**. Przy każdym restarcie robi `docker pull` i pobiera tylko zmienione warstwy; żeby przejąć nowy obraz, zrestartuj aplikację ręcznie lub włącz **ciągłe wdrażanie (continuous deployment)**, w którym webhook restartuje aplikację:

```azurecli
az webapp deployment container config \
    --resource-group myResourceGroup \
    --name myDocumentProcessor \
    --enable-cd true
```

Polecenie zwraca adres **webhooka**; w ACR utwórz webhook wywoływany przy zdarzeniu push (egzamin je lubi).

Kiedy App Service pobiera obraz:

- **pierwsze wdrożenie** lub zmiana odwołania do obrazu: wszystkie warstwy,
- **restart**: tylko zmienione warstwy (niezmienione warstwy z cache),
- **skalowanie w poziomie (scale out)**: każda nowa instancja pobiera obraz, czasem całość,
- **zmiana poziomu cenowego**: może przydzielić nową infrastrukturę, która pobiera obraz od zera, co wydłuża start. [docs: custom-container] zaznacza, że instancje mogą też zmienić się rzadko bez operacji skalowania.

Weryfikacja:

```azurecli
az webapp show \
    --resource-group myResourceGroup \
    --name myDocumentProcessor \
    --query defaultHostName \
    --output tsv
```

Otwórz adres w przeglądarce lub wywołaj `curl`; gdy kontener się nie uruchamia, użyj narzędzi z części D.

---

# CZĘŚĆ B - ZACHOWANIE KONTENERA W RUNTIME

## B1. Polecenie startowe
_Źródło: [unit 3]_

App Service uruchamia kontener według `ENTRYPOINT` i `CMD` z Dockerfile. **Polecenie startowe (startup command)** nadpisuje te domyślne wartości, gdy chcesz przekazać argumenty zależne od środowiska, uruchomić skrypt inicjalizujący (np. migracje bazy), wystartować wiele procesów albo zmienić domyślną konfigurację frameworka.

```azurecli
az webapp config set \
    --resource-group myResourceGroup \
    --name myDocumentProcessor \
    --startup-file "gunicorn --bind=0.0.0.0:8000 --workers=4 app:application"
```

Polecenie startowe zastępuje instrukcję `CMD`; `ENTRYPOINT` zostaje, chyba że zmienisz konfigurację kontenera bezpośrednio. Gdy potrzebna jest powłoka, poprzedź polecenie powłoką:

```azurecli
az webapp config set \
    --resource-group myResourceGroup \
    --name myDocumentProcessor \
    --startup-file "/bin/bash -c 'python migrate.py && gunicorn app:application'"
```

Objaśnienie: `gunicorn` to produkcyjny serwer WSGI dla Pythona (WSGI = standard łączący serwer z aplikacją, np. Flask); `&&` uruchamia drugie polecenie dopiero po sukcesie pierwszego.

## B2. Port kontenera (`WEBSITES_PORT`)
_Źródło: [unit 3] + [docs: https://learn.microsoft.com/en-us/azure/app-service/configure-custom-container] + [docs: https://learn.microsoft.com/en-us/dotnet/core/compatibility/containers/8.0/aspnet-port]_

App Service musi wiedzieć, na którym porcie **wewnątrz kontenera** są przyjmowane żądania HTTP. Według dokumentacji domyślnie zakłada port **80**. Jeśli kontener nasłuchuje na innym porcie, ustaw **ustawienie aplikacji** (app setting) `WEBSITES_PORT`:

```azurecli
az webapp config appsettings set \
    --resource-group myResourceGroup \
    --name myDocumentProcessor \
    --settings WEBSITES_PORT=8000
```

Platforma kończy TLS (TLS = szyfrowanie HTTPS) przed kontenerem, więc kontener dostaje zwykły HTTP także przy połączeniu HTTPS. App Service obsługuje **tylko jeden port** dla HTTP do kontenera niestandardowego. Pułapka w nazwach (egzamin je lubi): to ustawienie nazywa się `WEBSITES_PORT` (liczba mnoga), a nie `WEBSITE_PORT`.

Typowe porty domyślne frameworków (z modułu):

| Framework | Port domyślny | Ustawienie |
|---|---|---|
| Node.js (Express) | 3000 | `WEBSITES_PORT=3000` |
| Python (Gunicorn) | 8000 | `WEBSITES_PORT=8000` |
| Java (Spring Boot) | 8080 | `WEBSITES_PORT=8080` |
| ASP.NET Core | 80 | brak zmian (wg modułu) |

> **UWAGA - błąd w materiale Microsoftu** (unit 3): tabela podaje dla ASP.NET Core port domyślny 80 i "brak zmian" -> od .NET 8 oficjalne obrazy kontenerów ASP.NET Core nasłuchują domyślnie na porcie **8080**, a App Service domyślnie zakłada port 80, więc taka aplikacja wymaga `WEBSITES_PORT=8080` (albo ustawienia portu aplikacji na 80 przez `ASPNETCORE_HTTP_PORTS=80`). Źródło: https://learn.microsoft.com/en-us/dotnet/core/compatibility/containers/8.0/aspnet-port oraz https://learn.microsoft.com/en-us/azure/app-service/configure-custom-container.

> Uwaga o dokumentacji: moduł (unit 3) pisze, że App Service sam kieruje ruch, gdy kontener nasłuchuje na 80 lub 8080. Strona o kontenerach niestandardowych [docs: custom-container] mówi o domyślnym porcie 80 (wzmianka o 80 i 8080 dotyczy w niej opcji Docker Compose). Na egzaminie i w praktyce ustawiaj `WEBSITES_PORT` jawnie, gdy port jest inny niż 80.

## B3. Trwały magazyn (`WEBSITES_ENABLE_APP_SERVICE_STORAGE`)
_Źródło: [unit 3] + [docs: https://learn.microsoft.com/en-us/azure/app-service/configure-custom-container]_

Domyślnie zapisy do systemu plików kontenera są **ulotne (ephemeral)**: giną po restarcie lub przeniesieniu aplikacji na inną infrastrukturę. App Service może zamontować **trwały magazyn** w katalogu `/home`. Dla niestandardowych kontenerów **Linux** jest on **domyślnie wyłączony**; włączasz go ustawieniem:

```azurecli
az webapp config appsettings set \
    --resource-group myResourceGroup \
    --name myDocumentProcessor \
    --settings WEBSITES_ENABLE_APP_SERVICE_STORAGE=true
```

Po włączeniu: `/home` przetrwa restarty; wszystkie instancje skalowanej aplikacji współdzielą zawartość `/home`; w `/home/LogFiles` są logi kontenera i aplikacji. Zapisuj trwałe dane w `/home` lub podkatalogu (np. `/home/output/`). Pojemność zależy od planu App Service, a limit jest wspólny dla wszystkich aplikacji w planie; dla dużych wolumenów lub wysokiego I/O użyj zamontowanego Azure Storage. (Na Windows niestandardowe kontenery mają ten magazyn domyślnie włączony. [docs: custom-container])

## B4. Always On
_Źródło: [unit 3] + [docs: https://learn.microsoft.com/en-us/azure/app-service/configure-common]_

Aplikacja bez ruchu jest po ok. 20 minutach wyładowywana; następne żądanie powoduje **zimny start** (od kilku sekund do minut, zależnie od rozmiaru obrazu i inicjalizacji). **Always On** utrzymuje aplikację załadowaną:

```azurecli
az webapp config set \
    --resource-group myResourceGroup \
    --name myDocumentProcessor \
    --always-on true
```

Front-end wysyła wtedy żądanie `GET` do katalogu głównego aplikacji co pięć minut. [docs: configure-common] Wymaga poziomu cenowego **Basic lub wyższego** (egzamin je lubi). Zalecane dla: produkcji z wymogiem czasu odpowiedzi, długiego startu, dużych obrazów i usług z procesami w tle.

## B5. Kontrola stanu (health check)
_Źródło: [unit 3] + [docs: https://learn.microsoft.com/en-us/azure/app-service/monitor-instances-health-check]_

**Health check** wysyła żądania HTTP na wskazaną ścieżkę i restartuje (zastępuje) niezdrowe instancje:

```azurecli
az webapp config set \
    --resource-group myResourceGroup \
    --name myDocumentProcessor \
    --generic-configurations '{"healthCheckPath": "/health"}'
```

Aplikacja powinna mieć endpoint zwracający HTTP 200 przy zdrowiu, który może sprawdzać dostępność bazy, usług czy zasobów:

```python
@app.route('/health')
def health_check():
    return {'status': 'healthy'}, 200
```

```python
@app.route('/health')
def health_check():
    try:
        db.execute('SELECT 1')          # (1) sprawdź bazę
        storage.list_containers()       # (2) sprawdź magazyn
        return {'status': 'healthy'}, 200
    except Exception as e:
        return {'status': 'unhealthy', 'error': str(e)}, 503   # (3)
```

Objaśnienie: (1)-(2) sprawdzają zależności; (3) kod 5xx oznacza niezdrowy. Z dokumentacji [docs: health-check] (egzamin je lubi): ścieżka nie ma wartości domyślnej; ping co **1 minutę** na wszystkich instancjach; zdrowa odpowiedź to kod **200-299**; instancja jest uznana za niezdrową po **10** nieudanych żądaniach (wartość domyślna, konfigurowalna ustawieniem `WEBSITE_HEALTHCHECK_MAXPINGFAILURES` od 2 do 10) i usuwana z load balancera; ping jest nadal wysyłany i instancja wraca, gdy znów odpowie poprawnie; domyślnie nie więcej niż połowa instancji (`WEBSITE_HEALTHCHECK_MAXUNHEALTHYWORKERPERCENT`, domyślnie 50) jest wyłączana naraz; przy jednej instancji load balancer jej nie usuwa, ale po godzinie ciągłych niepowodzeń instancja jest zastępowana; do pełnego efektu potrzeba co najmniej dwóch instancji. Zmiana konfiguracji health check **restartuje aplikację**, więc na produkcji rób to ostrożnie. [unit 3]

---

# CZĘŚĆ C - USTAWIENIA APLIKACJI I SEKRETY

## C1. Ustawienia aplikacji (app settings)
_Źródło: [unit 4] + [docs: https://learn.microsoft.com/en-us/azure/app-service/configure-common]_

**App settings** to pary nazwa-wartość, które App Service wstrzykuje do kontenera jako **zmienne środowiskowe** przy starcie. Dzięki temu ten sam obraz działa w różnych środowiskach z różną konfiguracją, bez przebudowy.

```azurecli
az webapp config appsettings set \
    --resource-group myResourceGroup \
    --name myDocumentProcessor \
    --settings \
        STORAGE_ACCOUNT_NAME=mystorageaccount \
        LOG_LEVEL=INFO \
        MAX_DOCUMENT_SIZE_MB=50
```

```python
import os

storage_account = os.environ.get('STORAGE_ACCOUNT_NAME')                # (1)
log_level = os.environ.get('LOG_LEVEL', 'WARNING')                      # (2)
max_size = int(os.environ.get('MAX_DOCUMENT_SIZE_MB', 10))              # (3)
```

Objaśnienie: (1) odczyt zmiennej; (2) wartość domyślna, gdy brak; (3) wartości są tekstem, więc liczby trzeba rzutować. **Wszystkie** ustawienia są szyfrowane w spoczynku (encrypted at rest), niezależnie od tego, czy zawierają sekrety. Dla Linuksa zagnieżdżone klucze .NET z dwukropkiem zapisuj z podwójnym podkreśleniem: `ConnectionStrings:DefaultConnection` -> `ConnectionStrings__DefaultConnection` (dokumentacja dodaje: kropkę w nazwie zamień na pojedyncze podkreślenie). [docs: configure-common]

> **UWAGA - błąd w materiale Microsoftu** (unit 4): "App setting names can contain only letters, numbers, and underscores" -> nazwy mogą zawierać litery, cyfry, **kropki (.)** i podkreślenia (_). Źródło: https://learn.microsoft.com/en-us/azure/app-service/configure-common.

## C2. Connection strings
_Źródło: [unit 4]_

**Connection strings** to wyspecjalizowane ustawienia dla połączeń z bazami; App Service dodaje do nazwy zmiennej **prefiks typu**:

```azurecli
az webapp config connection-string set \
    --resource-group myResourceGroup \
    --name myDocumentProcessor \
    --connection-string-type SQLAzure \
    --settings DefaultConnection="Server=myserver.database.windows.net;Database=mydb;..."
```

| Typ | Prefiks zmiennej środowiskowej |
|---|---|
| SQL Server | `SQLCONNSTR_` |
| SQL Azure | `SQLAZURECONNSTR_` |
| MySQL | `MYSQLCONNSTR_` |
| PostgreSQL | `POSTGRESQLCONNSTR_` |
| Custom | `CUSTOMCONNSTR_` |

Dla przykładu powyżej zmienna to `SQLAZURECONNSTR_DefaultConnection`. W aplikacjach spoza .NET (Python, Node.js) prostsze są zwykłe app settings: prefiks tylko komplikuje, a nie daje korzyści frameworkom, które go nie oczekują. Connection strings są też szyfrowane w spoczynku. [docs: configure-common]

## C3. Edycja zbiorcza
_Źródło: [unit 4]_

Wyeksportuj ustawienia do JSON, edytuj i zaimportuj:

```azurecli
az webapp config appsettings list \
    --resource-group myResourceGroup \
    --name myDocumentProcessor \
    --output json > settings.json
```

```json
[
  { "name": "STORAGE_ACCOUNT_NAME", "value": "mystorageaccount", "slotSetting": false },
  { "name": "LOG_LEVEL", "value": "INFO", "slotSetting": false }
]
```

```azurecli
az webapp config appsettings set \
    --resource-group myResourceGroup \
    --name myDocumentProcessor \
    --settings @settings.json
```

Prefiks `@` oznacza odczyt z pliku. W portalu opcja **Environment variables** ma tryb **Advanced edit** z widokiem JSON.

## C4. Ustawienia slotów (slot settings)
_Źródło: [unit 4]_

**Sloty wdrożeń** pozwalają uruchomić różne wersje aplikacji obok siebie w tym samym planie. Każdy slot ma własną nazwę hosta i konfigurację, ale współdzieli zasoby obliczeniowe planu. Możesz **zamienić (swap)** sloty, żeby awansować staging na produkcję. Ustawienia oznaczone jako **slot settings** zostają przy slocie podczas swap (egzamin je lubi):

```azurecli
az webapp config appsettings set \
    --resource-group myResourceGroup \
    --name myDocumentProcessor \
    --slot staging \
    --slot-settings \
        ENVIRONMENT=staging \
        API_ENDPOINT=https://api-staging.example.com
```

Zastosowania: identyfikatory środowiska (`ENVIRONMENT=production` nie ma przejść na staging), adresy zależne od środowiska (API, bazy), flagi funkcji dostępne tylko w jednym środowisku, ustawienia diagnostyczne (szczegółowe logi w staging). Lista ustawień slotowych:

```azurecli
az webapp config appsettings list \
    --resource-group myResourceGroup \
    --name myDocumentProcessor \
    --query "[?slotSetting==\`true\`].name"
```

## C5. Odwołania do Key Vault
_Źródło: [unit 4] + [docs: https://learn.microsoft.com/en-us/azure/app-service/app-service-key-vault-references]_

Sekrety wymagające centralnego zarządzania, audytu lub automatycznej rotacji trzymaj w **Azure Key Vault** (magazyn sekretów). App Service pozwala w ustawieniu aplikacji wpisać **odwołanie do Key Vault**; aplikacja czyta zwykłą zmienną środowiskową i nie wie, że wartość pochodzi z Key Vault:

```azurecli
az webapp config appsettings set \
    --resource-group myResourceGroup \
    --name myDocumentProcessor \
    --settings \
        API_KEY="@Microsoft.KeyVault(SecretUri=https://myvault.vault.azure.net/secrets/api-key)"
```

Wymagania: **tożsamość zarządzana** włączona na aplikacji, nadanie jej dostępu do odczytu sekretów w Key Vault i składnia odwołania w wartości ustawienia. Bez wskazania wersji odwołanie rozwiązuje się do **najnowszej wersji** sekretu. Po rotacji sekretu App Service odświeża wartość w ciągu **24 godzin** (buforuje wartości i pobiera je ponownie co 24 h); każda zmiana konfiguracji aplikacji powoduje restart i natychmiastowe ponowne pobranie sekretów. Wymuszenie rozwiązania odwołań: uwierzytelnione żądanie POST do endpointu `.../config/configreferences/appsettings/refresh`. [docs: key-vault-references] Szczegóły Key Vault: moduł `manage-app-secrets-key-vault` (https://learn.microsoft.com/en-us/training/modules/manage-app-secrets-key-vault/).

## C6. Weryfikacja konfiguracji
_Źródło: [unit 4]_

Strona **SCM (Kudu)** pod `https://<nazwa-aplikacji>.scm.azurewebsites.net` -> **Environment** (lub bezpośrednio `.../Env`) pokazuje zmienne środowiskowe zastosowane do aplikacji: Twoje ustawienia i zmienne systemowe. Możesz też dodać endpoint diagnostyczny, który zwraca **niewrażliwe** wartości konfiguracji, albo sprawdzić logi startowe aplikacji.

---

# CZĘŚĆ D - OBSERWOWANIE I DIAGNOZOWANIE KONTENERA

## D1. Logi kontenera
_Źródło: [unit 5] + [docs: https://learn.microsoft.com/en-us/azure/app-service/troubleshoot-diagnostic-logs]_

App Service przechwytuje wyjście **stdout** i **stderr** kontenera. Włącz zapisywanie logów:

```azurecli
az webapp log config \
    --resource-group myResourceGroup \
    --name myDocumentProcessor \
    --docker-container-logging filesystem
```

Opcja `filesystem` zapisuje logi w systemie plików App Service (`/home/LogFiles/`). Logi obejmują: wyjście aplikacji (stdout), błędy (stderr), logi frameworka (start serwera WWW, żądania) i komunikaty platformy o cyklu życia kontenera. Aplikacja ma pisać sensowne logi na stdout; większość bibliotek logowania ma wyjście konsolowe.

## D2. Strumień logów (log stream)
_Źródło: [unit 5] + [docs: https://learn.microsoft.com/en-us/azure/app-service/troubleshoot-diagnostic-logs]_

Log stream pokazuje wyjście kontenera na żywo; przydaje się do diagnozy startu i obserwacji ruchu.

```azurecli
az webapp log tail \
    --resource-group myResourceGroup \
    --name myDocumentProcessor
```

Zakończ przez Ctrl+C. W portalu: aplikacja -> **Monitoring** -> **Log stream**. Strumień pokazuje wyjście ze wszystkich instancji, a wiersz ma identyfikator instancji do korelacji.

## D3. Konsola diagnostyczna Kudu (SCM)
_Źródło: [unit 5] + [unit 6-lab]_

**Kudu** (strona SCM: `https://<nazwa-aplikacji>.scm.azurewebsites.net`) to zaawansowana konsola diagnostyczna działająca jako osobna witryna; wymaga uwierzytelnienia poświadczeniami, które mogą zarządzać aplikacją. Przydatne funkcje:

- **Environment** - wszystkie zmienne środowiskowe kontenera (sprawdzisz, czy ustawienia zadziałały; widać też zmienne systemowe),
- **przeglądarka plików / konsola Debug** - pliki w zamontowanych ścieżkach, np. `/home` i `/home/LogFiles/`,
- **diagnostic dump** - archiwum ZIP z logami, konfiguracją i danymi diagnostycznymi (do analizy offline lub dla wsparcia),
- **ograniczenie:** SCM nie jest tym samym środowiskiem co kontener aplikacji, więc nie pokaże całego systemu plików ani procesów wewnątrz kontenera; do tego służy SSH.

## D4. Diagnostyka platformowa i Log Analytics
_Źródło: [unit 5]_

Do długoterminowego przechowywania i analizy wyślij logi przez **ustawienia diagnostyczne (diagnostic settings)** do Azure Monitor (Log Analytics), Event Hubs lub konta magazynu:

```azurecli
resourceId=$(az webapp show -g myResourceGroup -n myDocumentProcessor --query id -o tsv)
workspaceId=$(az monitor log-analytics workspace show -g myResourceGroup -n myLogAnalyticsWorkspace --query id -o tsv)

az monitor diagnostic-settings create \
    --resource "$resourceId" \
    --name myDiagnosticSettings \
    --workspace "$workspaceId" \
    --logs '[{"category":"AppServiceConsoleLogs","enabled":true},{"category":"AppServiceHTTPLogs","enabled":true}]'
```

| Kategoria | Opis |
|---|---|
| `AppServiceConsoleLogs` | stdout i stderr kontenera |
| `AppServiceHTTPLogs` | informacje o żądaniach i odpowiedziach HTTP |
| `AppServicePlatformLogs` | zdarzenia cyklu życia kontenera i komunikaty platformy |
| `AppServiceAppLogs` | logi na poziomie aplikacji (po konfiguracji) |

Zapytanie KQL (Kusto Query Language = język zapytań Log Analytics) o błędy z ostatniej godziny:

```kusto
AppServiceConsoleLogs
| where Level == "Error"
| where TimeGenerated > ago(1h)
| project TimeGenerated, ResultDescription
| order by TimeGenerated desc
```

Alerty i dashboardy budujesz na takich zapytaniach (KQL: moduł `analyze-telemetry-logs-metrics`).

## D5. Dostęp SSH do kontenera
_Źródło: [unit 5] + [docs: https://learn.microsoft.com/en-us/azure/app-service/configure-linux-open-ssh-session]_

Do interaktywnej diagnozy możesz połączyć się z działającym kontenerem przez SSH. Moduł podaje klasyczną konfigurację: obraz zawiera serwer OpenSSH (`openssh-server`), nasłuchuje na porcie **2222**, hasło użytkownika root to **`Docker!`** (wymagane przez App Service) i demon SSH startuje razem z aplikacją.

```dockerfile
RUN apt-get update && apt-get install -y openssh-server \
    && echo "root:Docker!" | chpasswd

COPY sshd_config /etc/ssh/

EXPOSE 8000 2222

CMD ["/bin/bash", "-c", "service ssh start && gunicorn app:application"]
```

Dostęp w portalu: aplikacja -> **Development Tools** -> **SSH**.

> Uwaga o dokumentacji: nowsze opcje (terminal w przeglądarce i `az webapp exec`, podgląd) łączą się przez Kudu jako uwierzytelniony proxy i **nie wymagają** serwera SSH w obrazie; klasyczne Kudu WebSSH i `az webapp ssh` mogą go nadal wymagać. Do terminala w przeglądarce potrzebny jest `/bin/bash` w obrazie; obrazy bez powłoki (distroless) nie otworzą sesji interaktywnej. [docs: custom-container] Przy SSH w obrazach niestandardowych `env` może pokazać tylko część zmiennych; do pełnej listy użyj widoku SCM `/Env`.

## D6. Typowe problemy i rozwiązania
_Źródło: [unit 5]_

**Kontener się nie uruchamia.** Objawy: adres zwraca błąd, logi pokazują błędy startu. Diagnoza: logi (log stream); czy obraz istnieje w rejestrze i poświadczenia są poprawne; czy kontener działa lokalnie z tą konfiguracją. Przyczyny: brak wymaganych zmiennych środowiskowych; niezgodność `WEBSITES_PORT` z portem, na którym nasłuchuje kontener; awaria aplikacji przy inicjalizacji (brak zależności).

**404 po wdrożeniu.** Kontener wygląda na uruchomiony, ale żądania zwracają 404. Diagnoza: `WEBSITES_PORT` zgodny z portem aplikacji; aplikacja nasłuchuje na `0.0.0.0`, a nie `localhost`; aplikacja obsługuje ścieżkę główną lub oczekiwane trasy. Przyczyny: nasłuch tylko na localhost, zły port, brak tras.

**Brakujące zmienne środowiskowe.** Aplikacja loguje brak konfiguracji. Diagnoza: ustawienia zapisane w portalu lub CLI; zmienne w Kudu (Environment); przy SSH użyj widoku SCM `/Env`, gdy wynik powłoki jest niepełny. Przyczyny: niezapisane zmiany, literówki w nazwach, aplikacja sprawdza zmienną, zanim App Service ją wstrzyknie.

**Wolne zimne starty.** Pierwsze żądania po bezczynności są wolniejsze. Diagnoza: rozmiar obrazu (`docker images`), czas startu w logach, ustawienie Always On. Rozwiązania: włącz Always On; zmniejsz obraz (mniejsze obrazy bazowe, budowa wieloetapowa - multi-stage builds); odrocz ciężką inicjalizację.

---

# CZĘŚĆ E - ĆWICZENIA

Przed ćwiczeniami przeczytaj części A-D. Układ ćwiczeń: Gdzie / Plik, kod lub kroki, Koszt, Zadania, Sprawdź, Sprzątanie. I1 odpowiada laboratorium z modułu (unit 6), ale używa mniejszej aplikacji podanej w całości (zamiast archiwum ze starterem); I2 i I3 są autorskie i korzystają z zasobów z I1 (zmienne `RG`, `ACR_NAME`, `APP_PLAN`, `APP_NAME`). Wszystkie ćwiczenia tworzą płatne zasoby; wykonaj Sprzątanie z ostatniego ćwiczenia, które robisz.

> Niesprawdzone: dokładne ceny rejestru ACR Basic i planu App Service B1 (Linux) za dzień - zweryfikuj w https://azure.microsoft.com/pricing/details/container-registry/ i https://azure.microsoft.com/pricing/details/app-service/linux/. Laboratorium Microsoftu używa planu P0v3 (droższy od B1) [unit 6-starter]; Always On wymaga Basic lub wyższego [docs: configure-common].

## I1. Wdrożenie kontenera z ACR do App Service z tożsamością zarządzaną - Python i C#
_Źródło: [unit 6] + [unit 6-lab] + [unit 6-starter] + [autor: niezweryfikowane]_
(rozwinięte w `02-deploy-containers-azure-app-service.md` §A4, §A5, §B2)

**Gdzie / Plik:** Cloud Shell (Bash), katalog `appsvc-lab`. Pliki aplikacji: `appsvc-lab/api/app.py`, `appsvc-lab/api/requirements.txt`, `appsvc-lab/api/Dockerfile`. Wymaga subskrypcji płatnej (ACR Tasks nie działają na darmowych kredytach).

Krok 1 - aplikacja (Python, Flask; konfiguracja ze zmiennych środowiskowych, zapis wyników do `/home/processed`):

```python
# appsvc-lab/api/app.py
import json
import os
import uuid

from flask import Flask, jsonify, request

app = Flask(__name__)

ENVIRONMENT = os.getenv("ENVIRONMENT", "development")            # (1)
STORAGE_PATH = os.getenv("STORAGE_PATH", "/home/processed")      # (2)


@app.route("/")
def root():
    return jsonify(service="docprocessor", environment=ENVIRONMENT)


@app.route("/health")
def health():
    return jsonify(status="healthy"), 200                         # (3)


@app.route("/process", methods=["POST"])
def process():
    text = request.get_data(as_text=True)
    doc_id = str(uuid.uuid4())
    result = {"id": doc_id, "words": len(text.split())}
    try:
        os.makedirs(STORAGE_PATH, exist_ok=True)
        with open(os.path.join(STORAGE_PATH, doc_id + ".json"), "w") as f:
            json.dump(result, f)                                  # (4)
        result["saved"] = True
    except OSError as e:
        result["saved"] = False
        result["error"] = str(e)
    return jsonify(result)


@app.route("/documents")
def documents():
    if not os.path.isdir(STORAGE_PATH):
        return jsonify(documents=[])
    names = [n[:-5] for n in os.listdir(STORAGE_PATH) if n.endswith(".json")]
    return jsonify(documents=names, count=len(names))
```

Objaśnienie: (1) środowisko z app setting `ENVIRONMENT`; (2) ścieżka zapisu, domyślnie w `/home`; (3) endpoint kontroli stanu; (4) zapis wyniku jako plik JSON w `STORAGE_PATH`. Aplikacja nie wie, czy `/home` jest trwały: to decyduje ustawienie platformy.

```text
# appsvc-lab/api/requirements.txt
flask==3.1.3
gunicorn==22.0.0
```

```dockerfile
# appsvc-lab/api/Dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app.py .
EXPOSE 8000
CMD ["gunicorn", "--bind", "0.0.0.0:8000", "--workers", "2", "--access-logfile", "-", "app:app"]
```

Objaśnienie: kontener nasłuchuje na porcie **8000** (nie 80), więc konieczne będzie `WEBSITES_PORT=8000`; `--access-logfile -` kieruje logi żądań na stdout.

Krok 2 - rejestr, obraz i plan:

```bash
az login
az provider register --namespace Microsoft.ContainerRegistry
az provider register --namespace Microsoft.Web
export RG=rg-appsvc-i1
export ACR_NAME=acrapp$RANDOM$RANDOM
export APP_PLAN=plan-docproc
export APP_NAME=docproc$RANDOM$RANDOM                                     # (1)
az group create --name $RG --location westeurope
az acr create --resource-group $RG --name $ACR_NAME --sku Basic          # (2)
az acr build --registry $ACR_NAME --image docprocessor:v1 ./api          # (3)
az appservice plan create --resource-group $RG --name $APP_PLAN --sku B1 --is-linux   # (4)
```

Objaśnienie: (1) nazwa aplikacji musi być globalnie unikalna (staje się częścią `<nazwa>.azurewebsites.net`); (2) rejestr z wyłączonym kontem admin (domyślnie); (3) build w chmurze, jak w module 01; (4) plan Linux w poziomie B1.

Krok 3 - aplikacja z tożsamością zarządzaną i rolą AcrPull:

```bash
az webapp create --resource-group $RG --plan $APP_PLAN --name $APP_NAME \
    --container-image-name $ACR_NAME.azurecr.io/docprocessor:v1          # (1)
az webapp identity assign --resource-group $RG --name $APP_NAME          # (2)
PRINCIPAL_ID=$(az webapp identity show --resource-group $RG --name $APP_NAME --query principalId --output tsv)
ACR_ID=$(az acr show --resource-group $RG --name $ACR_NAME --query id --output tsv)
az role assignment create --assignee $PRINCIPAL_ID --scope $ACR_ID --role AcrPull        # (3)
az webapp config set --resource-group $RG --name $APP_NAME \
    --acr-use-identity true --acr-identity '[system]'                    # (4)
az webapp config container set --resource-group $RG --name $APP_NAME \
    --container-image-name $ACR_NAME.azurecr.io/docprocessor:v1 \
    --container-registry-url https://$ACR_NAME.azurecr.io                # (5)
az webapp config appsettings set --resource-group $RG --name $APP_NAME \
    --settings WEBSITES_PORT=8000                                        # (6)
APP_URL=$(az webapp show --resource-group $RG --name $APP_NAME --query defaultHostName --output tsv)
echo "https://$APP_URL"
curl -s https://$APP_URL/                                                # (7)
```

Objaśnienie: (1) tworzy aplikację z obrazem z ACR; (2) włącza tożsamość system-assigned; (3) rola AcrPull ogranicza uprawnienia do pobierania (propagacja minutę-dwie); (4) App Service ma używać tożsamości zamiast danych admin; (5) jawnie ustawia obraz i adres rejestru (tu wskazujesz nowy tag przy aktualizacji); (6) port kontenera; (7) pierwsze żądanie jest wolniejsze, bo trzeba pobrać obraz. Może pojawić się komunikat o braku poświadczeń do ACR: to oczekiwane przy tożsamości zarządzanej (patrz A4).

**Odpowiednik C# (.NET).** Ten sam serwis w ASP.NET Core (minimal API). Różnica kluczowa: oficjalne obrazy ASP.NET Core od .NET 8 nasłuchują na **8080**, więc potrzebne jest `WEBSITES_PORT=8080` (patrz uwaga o błędzie w B2).

Plik `appsvc-lab/api-dotnet/DocProcessor.csproj`:

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
// appsvc-lab/api-dotnet/Program.cs
using System.Text.Json;

var app = WebApplication.CreateBuilder(args).Build();

string environment = Environment.GetEnvironmentVariable("ENVIRONMENT") ?? "development";   // (1)
string storage = Environment.GetEnvironmentVariable("STORAGE_PATH") ?? "/home/processed";   // (2)

app.MapGet("/", () => new { service = "docprocessor", environment });
app.MapGet("/health", () => Results.Ok(new { status = "healthy" }));                        // (3)

app.MapPost("/process", async (HttpRequest req) =>
{
    using var reader = new StreamReader(req.Body);
    string text = await reader.ReadToEndAsync();
    string id = Guid.NewGuid().ToString();
    int words = text.Split((char[]?)null, StringSplitOptions.RemoveEmptyEntries).Length;
    try
    {
        Directory.CreateDirectory(storage);
        await File.WriteAllTextAsync(Path.Combine(storage, id + ".json"),
            JsonSerializer.Serialize(new { id, words }));                                     // (4)
        return Results.Ok(new { id, words, saved = true });
    }
    catch (IOException e)
    {
        return Results.Ok(new { id, words, saved = false, error = e.Message });
    }
});

app.MapGet("/documents", () => Directory.Exists(storage)
    ? Directory.GetFiles(storage, "*.json").Select(f => Path.GetFileNameWithoutExtension(f))
    : Enumerable.Empty<string>());

app.Run();
```

```dockerfile
# appsvc-lab/api-dotnet/Dockerfile
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src
COPY DocProcessor.csproj .
RUN dotnet restore
COPY . .
RUN dotnet publish -c Release -o /out --no-restore

FROM mcr.microsoft.com/dotnet/aspnet:10.0
WORKDIR /app
COPY --from=build /out .
ENV STORAGE_PATH=/home/processed
EXPOSE 8080
CMD ["dotnet", "DocProcessor.dll"]
```

```bash
az acr build --registry $ACR_NAME --image docprocessor-dotnet:v1 ./api-dotnet
az webapp config container set --resource-group $RG --name $APP_NAME \
    --container-image-name $ACR_NAME.azurecr.io/docprocessor-dotnet:v1 \
    --container-registry-url https://$ACR_NAME.azurecr.io               # (1) zmiana obrazu
az webapp config appsettings set --resource-group $RG --name $APP_NAME --settings WEBSITES_PORT=8080   # (2)
curl -s https://$APP_URL/
```

Objaśnienie: (1) wskazanie innego obrazu restartuje aplikację (ta sama procedura jak przy nowym tagu); (2) port .NET zamiast 8000. Różnice względem Pythona: budowa wieloetapowa, port 8080, `Environment.GetEnvironmentVariable` zamiast `os.getenv`. Wersje obrazów MCR (`dotnet/sdk:10.0`, `dotnet/aspnet:10.0`) istnieją (sprawdzone 2026-11-22), podobnie `python:3.12-slim` w Docker Hub; aplikacje skompilowano/sprawdzono składniowo lokalnie 2026-11-22 [autor: niezweryfikowane] (nie uruchamiano w Azure). Aby wrócić do Pythona, ustaw obraz `docprocessor:v1` i `WEBSITES_PORT=8000`.

**Koszt:** rejestr ACR Basic (płatny za dzień) i plan App Service B1 Linux (płatny za godzinę działania, niezależnie od ruchu); buildy ACR Tasks naliczane za czas obliczeń (ceny: ostrzeżenie "Niesprawdzone" na początku części E). Przy pracy rzędu godziny koszt jest niewielki, o ile wykonasz Sprzątanie. [unit 6]

**Zadania:** (a) wdróż wersję Pythona i odczytaj `/`; (b) sprawdź, że aplikacja ma tożsamość i rolę AcrPull (`az role assignment list --assignee $PRINCIPAL_ID --scope $ACR_ID --output table`); (c) przełącz na obraz C# z `WEBSITES_PORT=8080`; (d) **zepsuj to:** przy obrazie Pythona usuń ustawienie portu (`az webapp config appsettings delete --resource-group $RG --name $APP_NAME --setting-names WEBSITES_PORT`) i wywołaj `/`.

**Sprawdź:** (a) odpowiedź JSON z `"service": "docprocessor"` i `"environment": "development"`; (b) lista zawiera rolę `AcrPull`; (c) odpowiedź JSON z tą samą treścią (z obrazu C#); (d) bez `WEBSITES_PORT` App Service zakłada port 80, kontener słucha na 8000, więc żądania nie dochodzą do aplikacji (błąd 5xx lub komunikat o błędzie aplikacji; dokładna treść - [autor: niezweryfikowane]). Przywróć `WEBSITES_PORT=8000` i potwierdź, że działa.

**Sprzątanie:**
```bash
az group delete --name rg-appsvc-i1 --yes --no-wait
az group exists --name rg-appsvc-i1     # po kilku minutach powinno wypisać false
```
Usunięcie grupy usuwa rejestr, plan i aplikację. Zasoby z I1 są używane w I2 i I3, więc wykonaj sprzątanie po ostatnim ćwiczeniu, które robisz.

## I2. Ustawienia aplikacji, trwały magazyn, Always On, health check i slot settings
_Źródło: [unit 3] + [unit 4] + [docs: https://learn.microsoft.com/en-us/azure/app-service/configure-common] + [autor: niezweryfikowane]_

**Gdzie / Plik:** Cloud Shell (Bash); zasoby z I1 (`RG`, `APP_NAME`, `APP_URL`), aplikacja Pythona (`docprocessor:v1`, `WEBSITES_PORT=8000`).

```bash
export RG=rg-appsvc-i1
export APP_NAME=$(az webapp list --resource-group $RG --query "[0].name" -o tsv)
export APP_URL=$(az webapp show --resource-group $RG --name $APP_NAME --query defaultHostName -o tsv)
az webapp config container set --resource-group $RG --name $APP_NAME \
    --container-image-name $(az acr list -g $RG --query "[0].loginServer" -o tsv)/docprocessor:v1   # (1)
az webapp config appsettings set --resource-group $RG --name $APP_NAME --settings WEBSITES_PORT=8000

# (2) ustawienia aplikacji
az webapp config appsettings set --resource-group $RG --name $APP_NAME \
    --settings ENVIRONMENT=production LOG_LEVEL=INFO
curl -s https://$APP_URL/                                               # widać "environment": "production"

# (3) zapis dokumentu i restart BEZ trwałego magazynu
echo "Azure App Service hosts containers" | curl -s -X POST https://$APP_URL/process --data-binary @-
curl -s https://$APP_URL/documents
az webapp restart --resource-group $RG --name $APP_NAME
sleep 60; curl -s https://$APP_URL/documents                            # lista pusta?

# (4) włącz trwały magazyn i powtórz
az webapp config appsettings set --resource-group $RG --name $APP_NAME \
    --settings WEBSITES_ENABLE_APP_SERVICE_STORAGE=true
echo "Persisted document" | curl -s -X POST https://$APP_URL/process --data-binary @-
az webapp restart --resource-group $RG --name $APP_NAME
sleep 60; curl -s https://$APP_URL/documents                            # dokument przetrwał?

# (5) Always On i health check
az webapp config set --resource-group $RG --name $APP_NAME --always-on true
az webapp config set --resource-group $RG --name $APP_NAME \
    --generic-configurations '{"healthCheckPath": "/health"}'
az webapp config show --resource-group $RG --name $APP_NAME --query "{alwaysOn:alwaysOn, healthCheckPath:healthCheckPath}" -o json

# (6) ustawienie "przyklejone" do slotu
az webapp config appsettings set --resource-group $RG --name $APP_NAME \
    --slot-settings ENVIRONMENT=production
az webapp config appsettings list --resource-group $RG --name $APP_NAME \
    --query "[?slotSetting==\`true\`].name" -o tsv
```

Objaśnienie: (1) przywraca obraz Pythona, jeśli robiłeś wariant C#; (2) app setting wstrzyknięty jako zmienna środowiskowa, bez przebudowy obrazu; (3) bez `WEBSITES_ENABLE_APP_SERVICE_STORAGE` zapis trafia do ulotnego systemu plików (aplikacja zgłosi `saved: true`, ale po restarcie dokument zniknie); (4) po włączeniu `/home` jest trwały; (5) Always On wymaga Basic+, health check restartuje aplikację przy zmianie; (6) `--slot-settings` bez `--slot` oznacza ustawienie produkcyjne jako slotowe (zostaje przy slocie podczas swap).

**Koszt:** brak nowych zasobów; zasoby z I1 nadal naliczają opłaty do czasu Sprzątania.

**Zadania:** (a) ustaw `ENVIRONMENT=production` i potwierdź w odpowiedzi `/`; (b) pokaż, że bez trwałego magazynu dokument ginie po restarcie; (c) włącz magazyn i pokaż, że dokument przetrwa restart; (d) włącz Always On i health check i odczytaj konfigurację; (e) oznacz `ENVIRONMENT` jako slot setting; (f) **zepsuj to:** ustaw `healthCheckPath` na nieistniejącą ścieżkę `/healthz` (`--generic-configurations '{"healthCheckPath": "/healthz"}'`) i obserwuj stan aplikacji przez kilka minut, potem przywróć `/health`.

**Sprawdź:** (a) `"environment": "production"`; (b) po restarcie lista `documents` jest pusta (lub nie zawiera dokumentu z (3)); (c) po restarcie lista zawiera dokument zapisany po włączeniu magazynu; (d) `alwaysOn: true`, `healthCheckPath: "/health"`; (e) polecenie `list` zwraca `ENVIRONMENT`; (f) ścieżka zwraca 404, więc ping nie dostaje kodu 200-299 i po kilkunastu nieudanych pingach instancja jest uznana za niezdrową; przy jednej instancji load balancer jej nie usuwa (patrz B5), ale status w portalu (Health check) pokazuje niezdrowy stan. [autor: niezweryfikowane]

**Sprzątanie:** przywróć `/health`: `az webapp config set --resource-group $RG --name $APP_NAME --generic-configurations '{"healthCheckPath": "/health"}'`. Jeśli to ostatnie ćwiczenie: `az group delete --name rg-appsvc-i1 --yes --no-wait`, potem `az group exists --name rg-appsvc-i1` (po kilku minutach: `false`).

## I3. Logi, strumień logów, Kudu i diagnoza błędnego portu
_Źródło: [unit 5] + [unit 6-lab] + [docs: https://learn.microsoft.com/en-us/azure/app-service/troubleshoot-diagnostic-logs] + [autor: niezweryfikowane]_

**Gdzie / Plik:** Cloud Shell (Bash) i przeglądarka; zasoby z I1 i I2.

```bash
export RG=rg-appsvc-i1
export APP_NAME=$(az webapp list --resource-group $RG --query "[0].name" -o tsv)
export APP_URL=$(az webapp show --resource-group $RG --name $APP_NAME --query defaultHostName -o tsv)

az webapp log config --resource-group $RG --name $APP_NAME --docker-container-logging filesystem   # (1)
az webapp log tail --resource-group $RG --name $APP_NAME                                            # (2) w drugim terminalu; Ctrl+C kończy
curl -s https://$APP_URL/; curl -s https://$APP_URL/health                                         # (3)
echo "Kudu: https://$APP_NAME.scm.azurewebsites.net"                                               # (4)
az webapp config appsettings list --resource-group $RG --name $APP_NAME --output table            # (5)

# (6) zepsuj: zły port
az webapp config appsettings set --resource-group $RG --name $APP_NAME --settings WEBSITES_PORT=9999
az webapp restart --resource-group $RG --name $APP_NAME
az webapp log tail --resource-group $RG --name $APP_NAME                                            # szukaj komunikatu o porcie
```

Objaśnienie: (1) włącza zapis stdout/stderr do `/home/LogFiles`; (2) strumień logów na żywo (przy skalowaniu w poziomie widać identyfikator instancji); (3) generuje wpisy w logach (gunicorn z `--access-logfile -` pisze żądania na stdout); (4) adres konsoli Kudu: otwórz go, wybierz **Environment**, sprawdź zmienne `ENVIRONMENT` i `WEBSITES_PORT`, a w **File Manager** katalog `/home/LogFiles/`; (5) lista ustawień (własne i systemowe); (6) zły port symuluje typową awarię z D6.

**Koszt:** brak nowych zasobów; logi w systemie plików mieszczą się w limicie magazynu planu.

**Zadania:** (a) włącz logowanie kontenera i obejrzyj strumień podczas kilku żądań; (b) w Kudu odszukaj `ENVIRONMENT=production` i pliki logów w `/home/LogFiles/`; (c) wypisz ustawienia aplikacji; (d) **zepsuj to:** ustaw `WEBSITES_PORT=9999`, zrestartuj aplikację i znajdź w logach przyczynę; (e) napraw (`WEBSITES_PORT=8000`) i potwierdź, że `/` działa.

**Sprawdź:** (a) w strumieniu pojawiają się wpisy żądań `GET /` i `GET /health`; (b) w widoku Environment jest `ENVIRONMENT=production`, a w `/home/LogFiles/` pliki logów; (c) w tabeli widać `WEBSITES_PORT`, `ENVIRONMENT` i ustawienia systemowe; (d) logi platformy informują, że kontener nie odpowiada na pingi HTTP na skonfigurowanym porcie 9999 (dokładny komunikat zależy od wersji platformy - [autor: niezweryfikowane]), a aplikacja jest niedostępna; (e) po przywróceniu portu `/` zwraca JSON.

**Sprzątanie:**
```bash
az group delete --name rg-appsvc-i1 --yes --no-wait
az group exists --name rg-appsvc-i1     # po kilku minutach: false
```

---

# CZĘŚĆ F - QUIZ Z MODUŁU MICROSOFTU (5 pytań) + wyjaśnienia

Learn pokazuje pytania, ale nie pokazuje poprawnych odpowiedzi, więc każda odpowiedź poniżej jest **wnioskowana** z treści jednostek.

## F1. Obraz nasłuchuje na porcie 8000, po wdrożeniu żądania kończą się błędami połączenia
-> *Ustaw ustawienie aplikacji `WEBSITES_PORT` na 8000.* (B2). Pułapki: zmiana `EXPOSE` w Dockerfile nie wpływa na to, gdzie App Service kieruje ruch; HTTP/2 nie ma związku z portem.
*(odpowiedź wnioskowana - Learn nie pokazuje odpowiedzi; uzasadnienie: unit 3, sekcja "Port configuration")*

## F2. Pliki wyjściowe znikają po restarcie kontenera
-> *Ustaw `WEBSITES_ENABLE_APP_SERVICE_STORAGE` na `true` i zapisuj pliki w `/home`.* (B3). Pułapki: większy obraz nie daje trwałości zapisów; Always On zapobiega zimnym startom, ale nie chroni systemu plików przed restartem.
*(odpowiedź wnioskowana - Learn nie pokazuje odpowiedzi; uzasadnienie: unit 3, sekcja "Persistent storage")*

## F3. Inny adres API dla staging i produkcji, który nie ma się zamienić podczas swap
-> *Skonfiguruj `API_ENDPOINT` jako slot setting.* (C4). Pułapki: adres w obrazie wymaga przebudowy dla każdego środowiska; connection strings nie rozwiązują problemu swap.
*(odpowiedź wnioskowana - Learn nie pokazuje odpowiedzi; uzasadnienie: unit 4, sekcja "Slot settings")*

## F4. Kontener startuje, ale health check zawodzi, a aplikacja ma endpoint `/healthz` zwracający 200
-> *Ścieżka health check w App Service jest skonfigurowana inaczej niż endpoint aplikacji.* (B5). Pułapki: health check nie wymaga HTTPS; ścieżka nie ma wartości domyślnej, więc trzeba ją ustawić jawnie.
*(odpowiedź wnioskowana - Learn nie pokazuje odpowiedzi; uzasadnienie: unit 3, sekcja "Health checks")*

## F5. Sprawdzenie, czy ustawienia aplikacji trafiły do działającego kontenera
-> *Strona Environment w konsoli diagnostycznej Kudu.* (C6, D3). Pułapki: log stream pokazuje wyjście kontenera, a nie listę zmiennych; metryki nie pokazują wartości ustawień.
*(odpowiedź wnioskowana - Learn nie pokazuje odpowiedzi; uzasadnienie: unit 4, sekcja "Verify configuration" i unit 5, sekcja "Diagnostic console (Kudu)")*

---

# CZĘŚĆ G - PYTANIA KONTROLNE (odpowiedz bez zaglądania)

## G1. Jakie dwie metody uwierzytelniania do ACR obsługuje App Service i którą zalecasz na produkcji?
<details><summary>Odpowiedź</summary>

Tożsamość zarządzana (zalecana na produkcji; wymaga roli AcrPull w rejestrze; system-assigned lub user-assigned) i dane administratora ACR (prostsze w development, ale hasło trafia do konfiguracji i wymaga ręcznej rotacji; trzeba włączyć konto admin).
</details>

## G2. Co się stanie, gdy wypchniesz nowy obraz pod ten sam tag, który App Service już wdrożył?
<details><summary>Odpowiedź</summary>

App Service tego nie wykryje. Obraz zostanie pobrany dopiero przy restarcie (pobierane są tylko zmienione warstwy); żeby nowy obraz był przejęty automatycznie, włącz ciągłe wdrażanie (`--enable-cd true`) i webhook w ACR. Lepsze rozwiązanie: nowy, unikalny tag i `az webapp config container set`.
</details>

## G3. Do czego służy `WEBSITES_PORT` i jaki jest port domyślny?
<details><summary>Odpowiedź</summary>

Wskazuje port wewnątrz kontenera, na który App Service kieruje ruch HTTP; domyślnie zakłada port 80. Obsługiwany jest tylko jeden port HTTP. Ustawiaj go, gdy kontener nasłuchuje gdzie indziej (np. 8000 dla Gunicorn, 8080 dla ASP.NET Core od .NET 8).
</details>

## G4. Co robi `WEBSITES_ENABLE_APP_SERVICE_STORAGE` i jaki jest domyślny stan na Linuksie?
<details><summary>Odpowiedź</summary>

Włącza trwały magazyn zamontowany w `/home`, współdzielony przez wszystkie instancje i przeżywający restarty. Dla niestandardowych kontenerów Linux domyślnie jest wyłączony (na Windows domyślnie włączony).
</details>

## G5. Jaki poziom cenowy jest potrzebny dla Always On i co to robi?
<details><summary>Odpowiedź</summary>

Basic lub wyższy. Aplikacja bez ruchu jest wyładowywana po ok. 20 minutach; Always On wysyła okresowe żądania (co 5 minut), aby ją utrzymać i uniknąć zimnych startów.
</details>

## G6. Jak działa health check w App Service?
<details><summary>Odpowiedź</summary>

Ping co minutę na ścieżkę, którą wskażesz (brak wartości domyślnej); zdrowa odpowiedź to 200-299; domyślnie po 10 nieudanych pingach instancja jest usuwana z load balancera (konfigurowalne od 2 do 10), a po dłuższym braku poprawy zastępowana. Zmiana konfiguracji restartuje aplikację.
</details>

## G7. Czym różnią się app settings od connection strings i które wybierzesz dla Pythona?
<details><summary>Odpowiedź</summary>

Obie są wstrzykiwane jako zmienne środowiskowe i szyfrowane w spoczynku, ale connection strings dostają prefiks typu (np. `SQLAZURECONNSTR_`). Dla aplikacji spoza .NET, np. Pythona, wybieraj zwykłe app settings.
</details>

## G8. Jak zapewnić, że ustawienie nie zamieni się między slotami podczas swap, i jak odwołać się do sekretu z Key Vault?
<details><summary>Odpowiedź</summary>

Oznacz je jako slot setting (`--slot-settings`). Do sekretu użyj w wartości ustawienia składni `@Microsoft.KeyVault(SecretUri=...)`; wymaga to tożsamości zarządzanej z dostępem do odczytu sekretów. Bez wersji odwołanie bierze najnowszą, a po rotacji wartość odświeża się w ciągu 24 godzin.
</details>

## G9. Wymień narzędzia diagnostyczne App Service i do czego służy każde.
<details><summary>Odpowiedź</summary>

Logi kontenera (`az webapp log config`, stdout/stderr w `/home/LogFiles`), log stream (`az webapp log tail`, na żywo), Kudu/SCM (Environment, pliki, diagnostic dump; nie pokazuje procesów w kontenerze), ustawienia diagnostyczne do Log Analytics (kategorie np. `AppServiceConsoleLogs`) z zapytaniami KQL oraz SSH do interaktywnej diagnozy kontenera.
</details>

---

# CZĘŚĆ H - CO DALEJ

## H1. Podsumowanie, następny krok i pokrycie egzaminu
_Źródło: [unit 8] + [docs: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/ai-200]_

**Podsumowanie modułu.** Wdrożyłeś własny kontener z ACR do App Service z uwierzytelnianiem tożsamością zarządzaną (bez zapisanych poświadczeń do rejestru), skonfigurowałeś zachowanie runtime (polecenie startowe, port, trwały magazyn), przekazałeś konfigurację specyficzną dla środowiska przez ustawienia aplikacji bez przebudowy obrazu oraz użyłeś logowania kontenera, log stream i Kudu do diagnozy. [unit 8]

**Co dalej:** moduł 03 - `deploy-sidecar-azure-app-service` (https://learn.microsoft.com/en-us/training/modules/deploy-sidecar-azure-app-service/): sidecary (kontenery pomocnicze obok głównego) na App Service, bezpieczne pobieranie obrazów, komunikacja lokalna, pliki współdzielone i diagnostyka. Wróć do A4 (tożsamość i AcrPull) i B2 (port) przy konfiguracji wielu kontenerów.

**Pokrycie egzaminu** - lista umiejętności AI-200 (stan study guide z 2026-10-30), domena 1, część "Implement container application hosting":

| Umiejętność (study guide) | Domena | Status | Gdzie |
|---|---|---|---|
| Build, store, version, and manage container images by using Azure Container Registry | 1 | uczone w module 01 | `01-store-manage-containers-azure-container-registry.md` §A1-A5, §C1-C8 |
| Build and run images by using Azure Container Registry Tasks | 1 | uczone w module 01 | `01-store-manage-containers-azure-container-registry.md` §B1-B6 |
| Deploy containers to Azure App Service, including configuring App Service to supply environment variables and secrets | 1 | uczone tutaj | `02-deploy-containers-azure-app-service.md` §A1-A6, §B1-B5, §C1-C6 (sekrety: §C5; więcej o Key Vault w module `manage-app-secrets-key-vault`) |

Pozostałe umiejętności egzaminu należą do innych modułów kursu; pełny przydział jest w `sources/_exam/skill-map.md`. Żaden punkt study guide nie jest orphanem, więc plik luk egzaminu nie jest potrzebny.

## H2. Uczciwie o egzaminie
_Źródło: [autor]_

Moduł pokrywa umiejętność "Deploy containers to Azure App Service, including configuring App Service to supply environment variables and secrets" w części dotyczącej App Service: źródła obrazu, tożsamość zarządzana i AcrPull, port, trwały magazyn, Always On, health check, app settings, slot settings, odwołania do Key Vault i diagnostyka. Pełne zarządzanie sekretami (rotacja, odczyt przez SDK) uczy osobny moduł `manage-app-secrets-key-vault`, a App Configuration - `manage-app-settings-app-config`. Na egzaminie spodziewaj się pytań o: nazwy ustawień (`WEBSITES_PORT`, `WEBSITES_ENABLE_APP_SERVICE_STORAGE`), wymagania (Always On = Basic+), rolę AcrPull, zachowanie przy tym samym tagu i slot settings. Moduł nie ma kodu SDK; kod aplikacji w ćwiczeniach jest autorski. Błędy w materiale Microsoftu opisane w tym pliku (B2, C1) sprawdź w dokumentacji, zanim zaufasz przykładom z modułu. Nie przerobiłeś jeszcze: sidecarów (moduł 03), Container Apps i AKS.
