# Moduł 07: Bezpieczne uruchamianie kodu generowanego przez AI w sesjach dynamicznych Azure Container Apps (pule sesji, interpreter kodu, identyfikatory sesji, błędy wykonania)

Moduł Microsoft Learn: **Run AI-generated code securely in Azure Container Apps dynamic sessions** - https://learn.microsoft.com/en-us/training/modules/dynamic-sessions-azure-container-apps/
Cel: po tym module rozumiesz, po co izoluje się kod generowany przez model AI, umiesz wybrać między wbudowanym interpreterem kodu a pulą z własnym kontenerem, skonfigurować pulę (pojemność, cooldown, ruch wychodzący), wykonać kod i wymienić pliki przez REST z identyfikatorem sesji oraz obsłużyć wygaśnięcie sesji, limity czasu i błędy wykonania.
Czas: ok. 3-4 h, orientacyjnie (moduł Learn ma ok. 84 min wg katalogu, w tym ćwiczenie ok. 30 min; tutaj dochodzą eksperymenty i ćwiczenie z SDK zarządzania). Wymagania: Azure CLI, Python 3.12, konto z prawem tworzenia puli sesji i przypisywania ról, region obsługujący sesje dynamiczne. Koszt: ćwiczenia I1-I2 tworzą pulę sesji (płatną za czas trwania sesji) - szczegóły w polach **Koszt**.

Źródła: `[unit N]` = jednostka modułu Learn, `[docs: URL]` = dokumentacja produktu, `[autor]` = dodane przez autora; pełna legenda: przewodnik stylu §3. "Moduł N" znaczy plik tutoriala `0N-*.md`.

> Uwaga o zakresie egzaminu: study guide AI-200 (stan z 2026-10-30) nie ma osobnej umiejętności o sesjach dynamicznych, ale moduł należy do kursu AI-200T00 i sąsiaduje z umiejętnościami domeny 1 (Container Apps). Traktuj go jako wiedzę pokrewną i uzupełniającą.

---

<!-- toc -->
**Spis treści**

- **[CZĘŚĆ A - GRANICA WYKONANIA I WYBÓR PULI](#część-a---granica-wykonania-i-wybór-puli)**
  - [A1. Po co granica wykonania i jak działa pula sesji](#a1-po-co-granica-wykonania-i-jak-działa-pula-sesji)
  - [A2. Wbudowany interpreter kodu (code interpreter)](#a2-wbudowany-interpreter-kodu-code-interpreter)
  - [A3. Integracja: REST bezpośrednio czy adapter frameworka](#a3-integracja-rest-bezpośrednio-czy-adapter-frameworka)
  - [A4. Pula z własnym kontenerem (custom container)](#a4-pula-z-własnym-kontenerem-custom-container)
  - [A5. Dopasowanie puli do zadania](#a5-dopasowanie-puli-do-zadania)
- **[CZĘŚĆ B - BEZPIECZNA KONFIGURACJA PULI](#część-b---bezpieczna-konfiguracja-puli)**
  - [B1. Pojemność puli i gotowe instancje](#b1-pojemność-puli-i-gotowe-instancje)
  - [B2. Cooldown (okres bezczynności) i cykl życia](#b2-cooldown-okres-bezczynności-i-cykl-życia)
  - [B3. Ruch wychodzący (egress) i tworzenie puli](#b3-ruch-wychodzący-egress-i-tworzenie-puli)
  - [B4. Tożsamości i dane w sesji](#b4-tożsamości-i-dane-w-sesji)
  - [B5. Praktyki dla produkcyjnych pul](#b5-praktyki-dla-produkcyjnych-pul)
- **[CZĘŚĆ C - WYKONYWANIE KODU I WYMIANA PLIKÓW](#część-c---wykonywanie-kodu-i-wymiana-plików)**
  - [C1. Uwierzytelnianie backendu](#c1-uwierzytelnianie-backendu)
  - [C2. Bezpieczne identyfikatory sesji](#c2-bezpieczne-identyfikatory-sesji)
  - [C3. Wysłanie kodu do wykonania](#c3-wysłanie-kodu-do-wykonania)
  - [C4. Wymiana plików z sesją](#c4-wymiana-plików-z-sesją)
  - [C5. Ponowne użycie sesji w wielu operacjach](#c5-ponowne-użycie-sesji-w-wielu-operacjach)
  - [C6. Integracja z agentem LangChain](#c6-integracja-z-agentem-langchain)
- **[CZĘŚĆ D - CYKL ŻYCIA I BŁĘDY WYKONANIA](#część-d---cykl-życia-i-błędy-wykonania)**
  - [D1. Efemeryczny stan sesji](#d1-efemeryczny-stan-sesji)
  - [D2. Kategorie błędów](#d2-kategorie-błędów)
  - [D3. Terminy wykonania (deadlines)](#d3-terminy-wykonania-deadlines)
  - [D4. Bezpieczne ponawianie (retry)](#d4-bezpieczne-ponawianie-retry)
  - [D5. Zwalnianie zakończonych sesji](#d5-zwalnianie-zakończonych-sesji)
  - [D6. Niezawodne operowanie przepływami sesji](#d6-niezawodne-operowanie-przepływami-sesji)
- **[CZĘŚĆ E - ĆWICZENIA](#część-e---ćwiczenia)**
  - [I1. Pula interpretera, rola Session Executor i klient REST w Pythonie (lab modułu)](#i1-pula-interpretera-rola-session-executor-i-klient-rest-w-pythonie-lab-modułu)
  - [I2. Egress i cooldown: eksperymenty z konfiguracją puli](#i2-egress-i-cooldown-eksperymenty-z-konfiguracją-puli)
  - [I3. Odczyt konfiguracji pul przez SDK zarządzania: Python i odpowiednik C# (.NET)](#i3-odczyt-konfiguracji-pul-przez-sdk-zarządzania-python-i-odpowiednik-c-net)
- **[CZĘŚĆ G - QUIZ Z MODUŁU MICROSOFTU (5 pytań) + wyjaśnienia](#część-g---quiz-z-modułu-microsoftu-5-pytań--wyjaśnienia)**
  - [G1. Zastrzeżony konwerter z natywnymi binariami - jaka pula?](#g1-zastrzeżony-konwerter-z-natywnymi-binariami---jaka-pula)
  - [G2. Niezaufany Python na wgranych plikach - jak ograniczyć ryzyko wycieku danych?](#g2-niezaufany-python-na-wgranych-plikach---jak-ograniczyć-ryzyko-wycieku-danych)
  - [G3. Wielodostępny backend - jak zarządzać identyfikatorami sesji?](#g3-wielodostępny-backend---jak-zarządzać-identyfikatorami-sesji)
  - [G4. API zwraca sukces HTTP, ale Python zgłosił wyjątek - jak sklasyfikować?](#g4-api-zwraca-sukces-http-ale-python-zgłosił-wyjątek---jak-sklasyfikować)
  - [G5. Timeout po stronie backendu, nie wiadomo, czy kod miał skutki uboczne - najbezpieczniejsza polityka](#g5-timeout-po-stronie-backendu-nie-wiadomo-czy-kod-miał-skutki-uboczne---najbezpieczniejsza-polityka)
- **[CZĘŚĆ H - PYTANIA KONTROLNE (odpowiedz bez zaglądania)](#część-h---pytania-kontrolne-odpowiedz-bez-zaglądania)**
  - [H1. Co daje pula sesji i jaka jest granica izolacji?](#h1-co-daje-pula-sesji-i-jaka-jest-granica-izolacji)
  - [H2. Kiedy wbudowany interpreter, a kiedy własny kontener?](#h2-kiedy-wbudowany-interpreter-a-kiedy-własny-kontener)
  - [H3. Wymień limity i wartości puli, które warto znać.](#h3-wymień-limity-i-wartości-puli-które-warto-znać)
  - [H4. Jakiej roli i jakiego odbiorcy tokenu wymaga wywołanie API zarządzania puli?](#h4-jakiej-roli-i-jakiego-odbiorcy-tokenu-wymaga-wywołanie-api-zarządzania-puli)
  - [H5. Jak bezpiecznie obsługiwać identyfikatory sesji?](#h5-jak-bezpiecznie-obsługiwać-identyfikatory-sesji)
  - [H6. Gdzie lądują wgrane pliki i jak wygląda ścieżka pobrania?](#h6-gdzie-lądują-wgrane-pliki-i-jak-wygląda-ścieżka-pobrania)
  - [H7. Jak odróżnić transport failure od execution failure?](#h7-jak-odróżnić-transport-failure-od-execution-failure)
  - [H8. Kiedy wolno ponawiać żądanie wykonania kodu?](#h8-kiedy-wolno-ponawiać-żądanie-wykonania-kodu)
  - [H9. Co zwróci udane usunięcie sesji i po co je robić?](#h9-co-zwróci-udane-usunięcie-sesji-i-po-co-je-robić)
- **[CZĘŚĆ J - CO DALEJ](#część-j---co-dalej)**
  - [J1. Następny krok i pokrycie egzaminu](#j1-następny-krok-i-pokrycie-egzaminu)
  - [J2. Uczciwie o egzaminie](#j2-uczciwie-o-egzaminie)

<!-- /toc -->

# CZĘŚĆ A - GRANICA WYKONANIA I WYBÓR PULI

## A1. Po co granica wykonania i jak działa pula sesji
_Źródło: [unit 1] + [unit 2]_

Aplikacje AI często generują kod (analiza danych, obliczenia, pliki). **Kodu wygenerowanego przez model nie można z góry uznać za bezpieczny**: uruchomiony w kontenerze aplikacji mógłby wykraść poświadczenia, zużyć wspólne zasoby albo wpłynąć na innych użytkowników. Rozwiązanie z modułu to **sesje dynamiczne** (dynamic sessions) w Azure Container Apps: gotowe, izolowane i **efemeryczne** (krótkotrwałe) środowiska do wykonywania takiego kodu.

**Pula sesji** (session pool) zawiera wstępnie rozgrzane środowiska, które platforma przydziela w milisekundach. Każda przydzielona sesja działa we własnej piaskownicy izolowanej przez **Hyper-V** (wirtualizacja sprzętowa), która oddziela procesy i system plików od hosta i od innych sesji. Dzięki temu backend wykonuje kod, nie uruchamiając go we własnym procesie. [unit 2]

```text
User request
    |
    v
AI application ---- model or orchestration layer
    |
    | validated code, files, and application-generated identifier
    v
Dynamic session pool
    |
    v
Isolated session ---- execution result and generated files
```

Backend uwierzytelnia się do puli i zleca pracę w imieniu użytkownika; użytkownik końcowy **nigdy** nie dostaje tokenów dostępu do puli ani nie wybiera identyfikatorów mogących wskazać cudzą sesję. Sesja trzyma stan tymczasowy; trwałe dane wejściowe i wyniki aplikacja zapisuje poza pulą. Długo działający kontener aplikacji nie jest równoważną granicą (kod obok API konkuruje o CPU i pamięć i może sięgnąć do plików, zmiennych środowiskowych i poświadczeń), a osobna aplikacja kontenerowa na każde żądanie dodaje opóźnienia startu i zarządzanie cyklem życia, które pula już zapewnia. [unit 2]

Po module potrafisz: wyjaśnić izolację kodu AI, wybrać wbudowany interpreter lub pulę z własnym kontenerem, skonfigurować pojemność, cykl życia i ruch wychodzący, użyć identyfikatorów sesji do wykonywania kodu i wymiany plików oraz obsłużyć wygaśnięcie, limity i błędy. [unit 1]

(egzamin je lubi: sesje = izolacja Hyper-V + efemeryczność; backend steruje dostępem)

## A2. Wbudowany interpreter kodu (code interpreter)
_Źródło: [unit 2] + [docs: https://learn.microsoft.com/azure/container-apps/sessions-code-interpreter]_

**Interpreter kodu** to zarządzane przez platformę środowisko uruchomieniowe z gotowym API: nie budujesz obrazu ani usługi wykonującej kod. Aplikacja może przesyłać kod, wgrywać pliki wejściowe i pobierać pliki wynikowe przez API zarządzania puli. Wybierz go, gdy obsługiwany runtime wykona zadanie bez własnych zależności systemowych: analiza CSV, obliczenia, wizualizacje, krótkie skrypty proponowane przez agenta.

```http
POST <POOL_MANAGEMENT_ENDPOINT>/executions?api-version=<API_VERSION>&identifier=<SESSION_ID>
POST <POOL_MANAGEMENT_ENDPOINT>/files?api-version=<API_VERSION>&identifier=<SESSION_ID>
GET <POOL_MANAGEMENT_ENDPOINT>/files/<FILE_NAME>/content?api-version=<API_VERSION>&identifier=<SESSION_ID>
```

API jest celowo ograniczone: gdy potrzebujesz nieobsługiwanego języka, natywnego programu wykonywalnego, zastrzeżonej biblioteki lub własnego protokołu, wbudowany interpreter nie jest właściwym wyborem. [unit 2] Dokumentacja dodaje: wersja Pythona zawiera popularne pakiety (m.in. NumPy, pandas, scikit-learn); typy puli interpretera to `PythonLTS`, `NodeLTS` i `Shell`. [docs: https://learn.microsoft.com/azure/container-apps/sessions-code-interpreter] [docs: https://learn.microsoft.com/azure/container-apps/session-pool]

## A3. Integracja: REST bezpośrednio czy adapter frameworka
_Źródło: [unit 2]_

Po wyborze interpretera zdecyduj, czy aplikacja woła REST bezpośrednio, czy używa adaptera frameworka orkiestracji AI. Bezpośrednie API daje jawną kontrolę nad uwierzytelnianiem, identyfikatorami sesji, żądaniami wykonania i plikami. Adapter zmniejsza ilość kodu, gdy model narzędzi frameworka pasuje do projektu. Dla agenta **LangChain** pakiet `langchain-azure-dynamic-sessions` dostarcza `SessionsPythonREPLTool`, czyli narzędzie wykonujące Python w sesji interpretera bez własnego kodu REST. Adapter **nie tworzy innego typu puli ani innej granicy bezpieczeństwa**: używa tych samych sesji, więc nadal potrzebna jest tożsamość z dostępem do puli i strategia instancji narzędzia oraz stanu na użytkownika. REST wybierz, gdy backend potrzebuje pełnej kontroli nad wymianą plików, odpowiedziami i mapowaniem identyfikatorów; LangChain, gdy istniejący agent potrzebuje głównie narzędzia do Pythona. Znajomość kontraktu REST przydaje się też do diagnozowania błędów, które raportuje adapter.

## A4. Pula z własnym kontenerem (custom container)
_Źródło: [unit 2] + [docs: https://learn.microsoft.com/azure/container-apps/session-pool]_

W puli **custom container** dostarczasz własny obraz. Platforma nadal daje pulę, szybki przydział, izolację Hyper-V i automatyczne sprzątanie, ale to **Twój obraz definiuje runtime i API** przyjmujące żądania. Wybierz ją dla specjalnych pakietów systemowych, zastrzeżonych narzędzi do dokumentów, języka niedostępnego w interpreterze albo usługi z własnym protokołem. Wymaga środowiska Container Apps z **profilami obciążeń** (workload profiles). Odpowiadasz za: **konserwację obrazu**, **ingress** aplikacji na skonfigurowanym porcie docelowym, **logowanie** na standardowe wyjście lub błędów oraz **sprawdzenia zdrowia** (sondy). [unit 2]

```json
{
  "properties": {
    "customContainerTemplate": {
      "containers": [
        {
          "name": "document-executor",
          "image": "contoso.azurecr.io/document-executor:1.0",
          "probes": [
            {
              "type": "Startup",
              "httpGet": {
                "path": "/ready",
                "port": 8080
              },
              "periodSeconds": 5,
              "failureThreshold": 30
            },
            {
              "type": "Liveness",
              "httpGet": {
                "path": "/health",
                "port": 8080
              },
              "periodSeconds": 10,
              "failureThreshold": 3
            }
          ]
        }
      ],
      "ingress": {
        "targetPort": 8080
      }
    }
  }
}
```
Sonda **Startup** pozwala runtime'owi z dłuższą inicjalizacją stać się gotowym; **Liveness** usuwa środowisko, które stało się niezdrowe po starcie. Żądania trafiają przez punkt końcowy zarządzania puli: ścieżka po punkcie końcowym (np. `/api/convert`) jest przekazywana na port docelowy kontenera. [unit 2]

Dokumentacja dodaje: pula custom container wspiera sondy **Liveness i Startup**; punkt końcowy zarządzania ma postać `https://<POOL>.<ENV_ID>.<REGION>.azurecontainerapps.io` i oprócz własnych ścieżek obsługuje `.management/getSession`, `.management/listSessions`, `.management/stopSession`; po zmianie obrazu zaktualizuj pulę **z nowym, unikalnym tagiem**, bo obraz jest buforowany w puli. [docs: https://learn.microsoft.com/azure/container-apps/session-pool]

## A5. Dopasowanie puli do zadania
_Źródło: [unit 2]_

Najlepsza pula to najmniej skomplikowana, która spełnia kontrakt wykonania. **Izolacja sama nie wymaga własnego obrazu**, bo oba typy dają izolowane sesje; o wyborze decydują zwykle wymagania runtime i protokołu.

| Scenariusz | Wybór |
|---|---|
| Analiza wgranego CSV | wbudowany interpreter Pythona (gdy pakiety wystarczą do analizy i wykresów) |
| Zastrzeżony konwerter z binariami natywnymi lub licencjonowanymi zależnościami | custom container |
| Izolowana przestrzeń robocza najemcy z własną aplikacją lub środowiskiem deweloperskim | custom container |
| Obliczenia z kodu agenta (krótkie fragmenty) | wbudowany interpreter (potrzebne tylko wyniki i pliki) |

Niezależnie od typu traktuj sesję jako **efemeryczną**: może zachować pliki i stan procesu dla powiązanych wywołań, ale cooldown w końcu usunie środowisko. Dokumenty źródłowe, zaakceptowane wyniki i zapisy audytowe trzymaj w trwałym magazynie aplikacji.

---

# CZĘŚĆ B - BEZPIECZNA KONFIGURACJA PULI

## B1. Pojemność puli i gotowe instancje
_Źródło: [unit 3] + [docs: https://learn.microsoft.com/azure/container-apps/session-pool]_

Ustawienie **maksymalnej liczby równoległych sesji** (max concurrent sessions) ogranicza, ile przydzielonych sesji może działać naraz. Zależy od przewidywanej współbieżności, nie od liczby zarejestrowanych użytkowników; to też granica bezpieczeństwa dla kosztów i zasobów zależnych. Przykład: jeśli 50 osób jest online, ale 10 naraz analizuje dokumenty, sensowny start to około 10. Test obciążeniowy pokaże, czy żądania czekają lub zawodzą po osiągnięciu limitu. W CLI parametr to `--max-sessions`, a **maksimum wynosi 600**. [unit 3] [docs: https://learn.microsoft.com/azure/container-apps/session-pool]

Pula z własnym kontenerem może też utrzymywać **gotowe instancje sesji** (`readySessionInstances`, w CLI `--ready-sessions`). Skracają opóźnienie przydziału, bo obraz i aplikacja już działają, ale utrzymują obliczenia także przy małym ruchu, więc najpierw zmierz czas startu i opóźnienie interaktywne. Fragment: dwie gotowe instancje i do 20 równoległych sesji; sondy pomagają zliczać jako gotowe tylko zdrowe środowiska. [unit 3]

```json
{
  "properties": {
    "scaleConfiguration": {
      "maxConcurrentSessions": 20,
      "readySessionInstances": 2
    },
    "customContainerTemplate": {
      "containers": [
        {
          "name": "document-executor",
          "image": "contoso.azurecr.io/document-executor:1.0",
          "probes": [
            {
              "type": "Startup",
              "httpGet": {
                "path": "/ready",
                "port": 8080
              },
              "periodSeconds": 5,
              "failureThreshold": 30
            },
            {
              "type": "Liveness",
              "httpGet": {
                "path": "/health",
                "port": 8080
              },
              "periodSeconds": 10,
              "failureThreshold": 3
            }
          ]
        }
      ],
      "ingress": {
        "targetPort": 8080
      }
    }
  }
}
```

## B2. Cooldown (okres bezczynności) i cykl życia
_Źródło: [unit 3] + [docs: https://learn.microsoft.com/azure/container-apps/session-pool]_

Cykl życia **timed** usuwa sesję po skonfigurowanym okresie bez żądań (**cooldown**); **każde żądanie zeruje licznik bezczynności**. Dłuższy cooldown daje wieloetapowej interakcji AI więcej czasu na ponowne użycie plików i stanu; krótszy szybciej zwalnia pojemność i usuwa dane tymczasowe. Dobierz go do przewidywanej przerwy między powiązanymi operacjami (analiza dokumentu może wymagać kilku minut na obejrzenie wyniku i prośbę o kolejny wykres). Nie traktuj cooldownu jako trwałej retencji: sesja może zniknąć po bezczynności albo po awarii. Dla niezaufanego kodu krótsza retencja zwykle zmniejsza ryzyko; źródło dokumentu i zaakceptowane wyniki zapisuj w magazynie aplikacji, żeby po nowej, pustej sesji odtworzyć dane wejściowe. [unit 3]

Wartości z dokumentacji (egzamin je lubi): `--cooldown-period` **300-3600 s**, domyślnie **300**; typ cyklu życia `Timed` (domyślny) lub `OnContainerExit` (dla własnego kontenera: sesja działa, aż kontener zakończy działanie lub minie `--max-alive-period`, domyślnie 3600 s; `cooldownPeriodInSeconds` nie jest wtedy wspierany). [docs: https://learn.microsoft.com/azure/container-apps/session-pool]

## B3. Ruch wychodzący (egress) i tworzenie puli
_Źródło: [unit 3] + [docs: https://learn.microsoft.com/azure/container-apps/session-pool]_

Sesje **domyślnie blokują ruch wychodzący**. To uniemożliwia wygenerowanemu kodowi pobieranie dodatkowych programów, wysyłanie wgranej treści na zewnętrzny adres lub użycie sesji do ataku typu DoS. Zostaw egress wyłączony, jeśli zadanie działa tylko na dostarczonym kodzie i plikach. Włączaj tylko przy konkretnej zależności zewnętrznej i akceptowalnym ryzyku. **Egress to ustawienie całej puli**, więc dotyczy każdej sesji w niej; lepiej rozdzielić obciążenia zaufane (z dostępem do sieci) od niezaufanych, niż osłabiać wspólną pulę.

```azurecli
az containerapp sessionpool create \
    --name ai-document-sessions \
    --resource-group <RESOURCE_GROUP> \
    --location <LOCATION> \
    --container-type PythonLTS \
    --max-sessions 20 \
    --cooldown-period 300 \
    --network-status EgressDisabled
```
Punkt końcowy zarządzania puli odczytasz po utworzeniu; aplikacja używa go do wykonywania kodu i pracy z plikami:
```azurecli
az containerapp sessionpool show \
    --name ai-document-sessions \
    --resource-group <RESOURCE_GROUP> \
    --query "properties.poolManagementEndpoint" \
    --output tsv
```
Dla puli interpretera punkt końcowy ma postać `https://<REGION>.dynamicsessions.io/subscriptions/<SUBSCRIPTION_ID>/resourceGroups/<RESOURCE_GROUP>/sessionPools/<POOL>`. Parametry `--container-type`: `PythonLTS` (domyślny), `NodeLTS`, `Shell`, `CustomContainer`; `--network-status`: `EgressDisabled` (domyślny) lub `EgressEnabled`. [docs: https://learn.microsoft.com/azure/container-apps/session-pool]

> Wersje sprawdzone 2026-11-22 (źródło: strony session-pool i sessions-code-interpreter w docs, `az containerapp sessionpool -h`): komendy `az containerapp sessionpool create|show|update|list|delete` istnieją w rozszerzeniu `containerapp` (1.3.0b6); w lab Microsoftu rozszerzenie instalowane jest z `--allow-preview true`; parametr `--probe-yaml` jest oznaczony jako Preview. Przed użyciem sprawdź https://learn.microsoft.com/azure/container-apps/session-pool.

## B4. Tożsamości i dane w sesji
_Źródło: [unit 3]_

**Izolacja Hyper-V oddziela sesje od siebie, ale kod w jednej sesji ma dostęp do plików, zmiennych środowiskowych i dostępnych w niej poświadczeń.** Umieszczaj sekret lub dokument w sesji tylko wtedy, gdy zezwalasz kodowi sesji na dostęp do tych danych: izolacja między sesjami nie czyni danych bezpiecznymi przed kodem, który sam wykonujesz.

Tożsamość backendu i tożsamość sesji mają inne cele. Backend używa roli **Azure ContainerApps Session Executor**, by wołać API zarządzania puli; użytkownicy końcowi nigdy nie dostają tokenu tej tożsamości, a wygenerowany kod jej nie potrzebuje. Pula z własnym kontenerem może mieć **tożsamość zarządzaną** do pobierania obrazu lub dostępu do zasobów chronionych przez Entra. Dostęp w czasie działania jest ryzykowniejszy, bo niezaufany kod w sesji może zażądać tokenów tej tożsamości; jeśli jest nieunikniony, użyj osobnej tożsamości o najmniejszym zakresie, nie szerszej tożsamości backendu.

## B5. Praktyki dla produkcyjnych pul
_Źródło: [unit 3]_

- **Rozdzielaj poziomy zaufania:** inne pule dla kodu o różnym egress, tożsamości lub dostępie do danych.
- **Minimalizuj dane w sesji:** wgrywaj tylko pliki potrzebne do bieżącego zadania, a zatwierdzone wyniki zapisuj gdzie indziej.
- **Ograniczaj współbieżność:** maksimum chroni koszt i systemy zależne przy skokach ruchu.
- **Preferuj wyłączony egress:** zezwalaj na ruch wychodzący tylko dla ocenionego, uzasadnionego obciążenia.
- **Używaj wąskich tożsamości:** tylko role i zakresy, których sesja lub backend potrzebuje.
- **Monitoruj wykorzystanie:** dla pul custom śledź liczbę sesji wykonywanych, tworzonych i gotowych.

---

# CZĘŚĆ C - WYKONYWANIE KODU I WYMIANA PLIKÓW

## C1. Uwierzytelnianie backendu
_Źródło: [unit 4] + [docs: https://learn.microsoft.com/azure/container-apps/sessions-code-interpreter]_

Każde żądanie do API zarządzania wymaga tokenu nośnika (bearer) Microsoft Entra. Tożsamość wywołująca potrzebuje roli **Azure ContainerApps Session Executor** na puli sesji, a **odbiorca tokenu (audience) musi mieć wartość `https://dynamicsessions.io`**. Tożsamość zarządzana daje uwierzytelnianie bez haseł, gdy backend działa w Azure. Rolę przypisz na zakresie puli, żeby tożsamość nie mogła wykonywać kodu w obcych pulach. [unit 4]

```azurecli
az role assignment create \
    --role "Azure ContainerApps Session Executor" \
    --assignee <BACKEND_PRINCIPAL_ID> \
    --scope <SESSION_POOL_RESOURCE_ID>
```
```python
from azure.identity import DefaultAzureCredential
credential = DefaultAzureCredential()
access_token = credential.get_token(
    "https://dynamicsessions.io/.default"
).token
```
`DefaultAzureCredential` wybiera poświadczenia deweloperskie lokalnie (np. po `az login`) i tożsamość zarządzaną w Azure, bez sekretu w kodzie; token pobieraj przy przygotowaniu wywołania, a nie cache'uj w nieskończoność (credential sam buforuje i odnawia). Dokumentacja dodaje, że tokeny generuje tożsamość z rolą Session Executor **lub Contributor** na puli. [docs: https://learn.microsoft.com/azure/container-apps/sessions-code-interpreter]

## C2. Bezpieczne identyfikatory sesji
_Źródło: [unit 4] + [docs: https://learn.microsoft.com/azure/container-apps/sessions-code-interpreter]_

**Identyfikator sesji** kieruje żądanie do istniejącej sesji albo powoduje przydział nowej. Ponowne użycie identyfikatora zachowuje pliki i stan między powiązanymi wywołaniami. Skoro w połączeniu z autoryzowanym tokenem daje dostęp do tego stanu, backend traktuje go jako **wrażliwy**. Generuj nieprzewidywalny identyfikator **po stronie serwera** dla każdego uwierzytelnionego użytkownika lub rozmowy z agentem, trzymaj powiązanie w stanie serwera i **nie przyjmuj dowolnego identyfikatora od klienta**. Losowy UUID się nadaje (jego postać z myślnikami spełnia format identyfikatora).

```python
from uuid import uuid4
def create_session_identifier() -> str:
    return str(uuid4())
```
Sam identyfikator nie uwierzytelnia wywołującego: backend nadal autoryzuje użytkownika przed użyciem zapisanego powiązania i uwierzytelnia każde żądanie do puli własnym tokenem; użytkownik nie może zmienić parametru żądania, by wskazać cudzą rozmowę. Dokumentacja opisuje identyfikator jako dowolny ciąg unikalny w puli (np. identyfikator użytkownika lub rozmowy) i podkreśla, że jest to informacja wrażliwa, a aplikacja musi dopilnować dostępu użytkowników tylko do własnych sesji. [docs: https://learn.microsoft.com/azure/container-apps/sessions-code-interpreter]

## C3. Wysłanie kodu do wykonania
_Źródło: [unit 4]_

Punkt końcowy `executions` przyjmuje kod inline i zwraca odpowiedź wykonania. W produkcyjnej aplikacji AI kod może dostarczyć warstwa orkiestracji, ale backend **waliduje żądanie względem własnej polityki** przed wysłaniem: izolacja zmniejsza skutki niezaufanego kodu, ale nie rozstrzyga, czy żądana akcja jest właściwa.

```python
from azure.identity import DefaultAzureCredential
API_VERSION = "2025-10-02-preview"
TOKEN_SCOPE = "https://dynamicsessions.io/.default"
class DynamicSessionClient:
    def __init__(
        self,
        pool_management_endpoint: str,
        session_identifier: str,
    ) -> None:
        self.endpoint = pool_management_endpoint.rstrip("/")
        self.identifier = session_identifier
        self.credential = DefaultAzureCredential()
    def _headers(self) -> dict[str, str]:
        token = self.credential.get_token(TOKEN_SCOPE).token
        return {"Authorization": f"Bearer {token}"}
    def _params(self) -> dict[str, str]:
        return {
            "api-version": API_VERSION,
            "identifier": self.identifier,
        }
```
```python
import requests
def execute_python(
    client: DynamicSessionClient,
    code: str,
) -> dict:
    response = requests.post(
        f"{client.endpoint}/executions",
        headers={
            **client._headers(),
            "Content-Type": "application/json",
        },
        params=client._params(),
        json={
            "properties": {
                "codeInputType": "inline",
                "executionType": "synchronous",
                "code": code,
            }
        },
        timeout=30,
    )
    response.raise_for_status()
    return response.json()
```
`raise_for_status()` sprawia, że błędy uwierzytelniania, walidacji, pojemności i usługi nie wyglądają na udane wykonania, ale **wywołujący musi jeszcze sprawdzić dane wykonania**, czy kod Pythona nie zgłosił błędu. Wersję API trzymaj w konfiguracji, nie rozproszoną po metodach, i testuj kontrakt odpowiedzi przy jej zmianie. Ciało żądania z `properties` odpowiada przykładowi w dokumentacji (`api-version=2025-10-02-preview`). [docs: https://learn.microsoft.com/azure/container-apps/sessions-code-interpreter]

## C4. Wymiana plików z sesją
_Źródło: [unit 4] + [docs: https://learn.microsoft.com/azure/container-apps/sessions-code-interpreter]_

Operacje na plikach pozwalają kodowi przetwarzać pliki użytkownika i tworzyć artefakty (wykresy, przekształcone dokumenty). Backend wgrywa **tylko pliki autoryzowane dla bieżącego zadania**. Sesje interpretera zapisują wgrane pliki w **`/mnt/data`**, gdzie kolejne wykonania z tym samym identyfikatorem mogą po nie sięgnąć.

```python
from pathlib import Path
import requests
def upload_file(
    client: DynamicSessionClient,
    file_path: Path,
) -> dict:
    with file_path.open("rb") as source:
        response = requests.post(
            f"{client.endpoint}/files",
            headers=client._headers(),
            params=client._params(),
            files={
                "file": (
                    file_path.name,
                    source,
                    "application/octet-stream",
                )
            },
            timeout=30,
        )
    response.raise_for_status()
    return response.json()
```
Po utworzeniu pliku przez kod backend może wylistować pliki sesji i pobrać wybrany artefakt. Nazwę pliku **koduj jako jeden segment ścieżki URL** (nazwy mogą zawierać znaki zarezerwowane) i **nigdy nie buduj lokalnej ścieżki docelowej z niezwalidowanej nazwy** (path traversal mógłby nadpisać niezamierzony plik).

```python
from urllib.parse import quote
def list_files(client: DynamicSessionClient) -> dict:
    response = requests.get(
        f"{client.endpoint}/files",
        headers=client._headers(),
        params=client._params(),
        timeout=15,
    )
    response.raise_for_status()
    return response.json()
def download_file(
    client: DynamicSessionClient,
    file_name: str,
    destination: Path,
) -> None:
    encoded_file_name = quote(file_name, safe="")
    response = requests.get(
        f"{client.endpoint}/files/{encoded_file_name}/content",
        headers=client._headers(),
        params=client._params(),
        timeout=30,
    )
    response.raise_for_status()
    destination.write_bytes(response.content)
```
**Limit wgrywania pliku w interpreterze: 128 MB** (przy przekroczeniu możliwy HTTP 413); dla większych danych zmień zadanie tak, by sesja dostawała mniejszy, zatwierdzony zestaw. [unit 4] [docs: https://learn.microsoft.com/azure/container-apps/sessions-code-interpreter] Lista plików w odpowiedzi ma postać `value[].properties.filename` (wraz z `size` i `lastModifiedTime`); dozwolone znaki nazw to m.in. litery, cyfry, `- _ . @ $ & = ; , # % ^ ( )` i znaki Unicode. [docs: https://learn.microsoft.com/azure/container-apps/sessions-code-interpreter]

## C5. Ponowne użycie sesji w wielu operacjach
_Źródło: [unit 4]_

Powiązane wywołania z tym samym identyfikatorem trafiają do tego samego środowiska, dopóki sesja nie wygaśnie: wgrywasz dokument raz, wykonujesz analizę i tworzysz kolejną wizualizację bez ponownego wgrywania. Inny identyfikator tworzy lub adresuje osobne środowisko.

```python
from pathlib import Path

session_id = create_session_identifier()
client = DynamicSessionClient(pool_endpoint, session_id)

upload_file(client, Path("operational-data.csv"))
execution = execute_python(client, supplied_python_payload)
available_files = list_files(client)
download_file(client, "trend.png", Path("trend.png"))
```
Ponowne użycie sesji to chwilowa wygoda, **nie gwarancja trwałego przechowywania**: zapisuj zaakceptowane wyniki i dane źródłowe w magazynie aplikacji, by odtworzyć stan po usunięciu sesji przez cooldown.

## C6. Integracja z agentem LangChain
_Źródło: [unit 4]_

Gdy aplikacja już używa LangChain, `langchain-azure-dynamic-sessions` dostarcza `SessionsPythonREPLTool` jako wyższy poziom integracji z tą samą pulą interpretera. Narzędzie losuje identyfikator sesji przy tworzeniu instancji, a wywołania przez tę instancję używają tej samej sesji, dopóki nie wygaśnie.

```python
# Code fragment - focus on creating the LangChain tool
from langchain_azure_dynamic_sessions import SessionsPythonREPLTool

python_repl = SessionsPythonREPLTool(
    pool_management_endpoint=pool_management_endpoint
)
agent_tools = [python_repl]
```
**Nie rejestruj jednej długo żyjącej instancji narzędzia jako singletona dla wszystkich użytkowników**: twórz osobne narzędzie i kontekst agenta na uwierzytelnionego użytkownika lub rozmowę, żeby niepowiązane żądania nie dzieliły plików ani stanu Pythona. Aplikacja nadal autoryzuje użytkowników, waliduje żądania agenta, przypisuje swojej tożsamości rolę Session Executor i przechowuje trwałe artefakty poza sesją. Adapter pasuje, gdy agent głównie potrzebuje Pythona jako narzędzia; bezpośrednie API daje więcej kontroli, gdy backend musi wgrać pliki przed wykonaniem, pobrać konkretne artefakty, zarządzać mapowaniem identyfikatorów lub interpretować pełną odpowiedź usługi.

> Wersje sprawdzone 2026-11-22 (źródło: pypi.org, api.nuget.org): `langchain-azure-dynamic-sessions` 1.0.4 (2026-09-24, wymaga Pythona >= 3.11), `requests` 2.34.2, `azure-identity` 1.27.0; API danych: `executions` i `files` z `api-version=2025-10-02-preview`, `session` (get/delete) z `2025-02-02-preview` (jak w dokumentacji code interpreter). Wersje API zmieniają się; przed użyciem sprawdź https://learn.microsoft.com/azure/container-apps/sessions-code-interpreter.

---

# CZĘŚĆ D - CYKL ŻYCIA I BŁĘDY WYKONANIA

## D1. Efemeryczny stan sesji
_Źródło: [unit 5] + [docs: https://learn.microsoft.com/azure/container-apps/sessions-code-interpreter]_

Sesja timed pozostaje przydzielona, dopóki otrzymuje żądania; każde żądanie zeruje licznik bezczynności. Po cooldownie bez aktywności Azure Container Apps **niszczy sesję i jej pliki tymczasowe**. Późniejsze żądanie z tym samym identyfikatorem może dostać **nową, pustą sesję**: identyfikator zachowuje routing tylko dopóki oryginalna sesja istnieje. Zaprojektuj przepływ tak, by rozpoznawał brak stanu tymczasowego: jeśli kod analizy zgłasza, że `/mnt/data/operational-data.csv` nie istnieje, backend pobiera autoryzowane źródło z trwałego magazynu, wgrywa je do nowej sesji i uruchamia ponownie idempotentny krok analizy.

Backend może zapytać o sesję (czas utworzenia, ostatni dostęp, wygaśnięcie): przydatne do diagnostyki i komunikatów, ale sesja może zniknąć między sprawdzeniem a następną operacją.

```http
GET <POOL_MANAGEMENT_ENDPOINT>/session?identifier=<SESSION_ID>&api-version=2025-02-02-preview
Authorization: Bearer <ACCESS_TOKEN>
```
Odpowiedź (`SessionView`) zawiera `identifier`, `etag`, `expiresAt`, `createdAt` i `lastAccessedAt`; `listSessions` listuje sesje stronami po 300 (paginacja `skip`). [docs: https://learn.microsoft.com/azure/container-apps/sessions-code-interpreter]

## D2. Kategorie błędów
_Źródło: [unit 5]_

**Sukces HTTP nie oznacza, że kod dał poprawny wynik biznesowy.** Transport może się udać, a Python zgłosić wyjątek albo zwrócić dane niespełniające kontraktu. Backend waliduje obie warstwy, zanim przekaże wynik warstwie orkiestracji. Cztery kategorie:

| Kategoria | Znaczenie |
|---|---|
| **Transport failure** | uwierzytelnianie, autoryzacja, błędne żądanie, pojemność, sieć lub usługa uniemożliwiły wykonanie operacji API |
| **Execution failure** | sesja uruchomiła ładunek, ale kod zgłosił wyjątek, przekroczył limit lub zwrócił nieudany wynik |
| **Validation failure** | kod się zakończył, ale wynik ma zły schemat, brakuje wymaganego artefaktu albo narusza regułę aplikacji |
| **Successful result** | transport OK, wykonanie bez błędu, wynik przeszedł walidację |

```python
from dataclasses import dataclass
from enum import StrEnum
from typing import Any

class ResultStatus(StrEnum):
    SUCCEEDED = "succeeded"
    TRANSPORT_ERROR = "transport_error"
    EXECUTION_ERROR = "execution_error"
    VALIDATION_ERROR = "validation_error"

@dataclass(frozen=True)
class ExecutionResult:
    status: ResultStatus
    output: dict[str, Any] | None
    message: str | None
    trace_id: str | None
```
Nie pokazuj użytkownikowi surowych stack trace'ów, tokenów ani wrażliwej treści; zwracaj wartość korelacyjną, by operator dopasował kontrolowaną odpowiedź do chronionych logów.

## D3. Terminy wykonania (deadlines)
_Źródło: [unit 5] + [docs: https://learn.microsoft.com/azure/container-apps/sessions-code-interpreter]_

Wykonania w interpreterze mają platformowy maksymalny czas działania: dokumentacja podaje **220 sekund** na jedno wykonanie (w CLI `--timeout-in-seconds` 1-220, domyślnie 60). Backend powinien jednak ustawiać **własny limit HTTP**, krótszy niż czas oczekiwania wołającego, żeby zdążyć zwalidować wynik i odpowiedzieć kontrolowanie; różne operacje mają różne limity (lista plików krócej niż analiza). **Limit klienta HTTP nie gwarantuje, że zdalne wykonanie zatrzyma się w tej samej chwili**, więc nie zakładaj, że operacja po timeoucie nie miała skutków; limit platformy i cykl życia sesji to osobne granice. Obsługuj `Timeout` osobno od błędów statusu HTTP, by telemetria odróżniała "przestałem czekać" od "usługa odrzuciła żądanie". Usługa zwraca identyfikator śledzenia (trace ID) w ciele błędu strukturalnego. [unit 5] [docs: https://learn.microsoft.com/cli/azure/containerapp/session/code-interpreter?view=azure-cli-latest]

```python
# Code fragment - focus on explicit timeout and error categories
import requests
try:
    response = requests.post(
        execution_url,
        headers=headers,
        params=params,
        json=payload,
        timeout=(5, 30),
    )
    response.raise_for_status()
except requests.Timeout as error:
    raise SessionExecutionTimeout(
        "The session didn't return a result before the application deadline."
    ) from error
except requests.HTTPError as error:
    try:
        error_body = error.response.json()
    except requests.JSONDecodeError:
        trace_id = None
    else:
        error_details = error_body.get("error")
        trace_id = (
            error_details.get("traceId")
            if isinstance(error_details, dict)
            else None
        )
    raise SessionTransportError(
        status_code=error.response.status_code,
        trace_id=trace_id,
    ) from error
```
Wyjątki `SessionExecutionTimeout` i `SessionTransportError` to typy aplikacyjne zdefiniowane gdzie indziej; produkcyjna implementacja obsłuży też błędy połączenia i zmapuje każdą awarię na standardową telemetrię i odpowiedzi.

## D4. Bezpieczne ponawianie (retry)
_Źródło: [unit 5]_

Ponawianie może pomóc przy przejściowych błędach sieci i usługi, ale **powtórzenie dowolnego wygenerowanego kodu może zdublować skutki uboczne**. Timeout odpowiedzi jest niejednoznaczny (wykonanie mogło się skończyć po tym, jak przestałeś czekać). Ponawiaj automatycznie tylko operacje **idempotentne**, gdy aplikacja potrafi zweryfikować wynik. Naturalni kandydaci: odczyty metadanych plików i zapytania o status sesji; deterministyczna analiza czytająca wejście i nadpisująca znany plik wyjściowy (po potwierdzeniu stanu). Nie powtarzaj automatycznie kodu wysyłającego wiadomości, zapisującego do systemów zewnętrznych lub tworzącego pliki o unikalnych nazwach.

```python
# Code fragment - focus on retry eligibility
TRANSIENT_STATUS_CODES = {429, 500, 502, 503, 504}

def can_retry(
    *,
    method: str,
    status_code: int,
    operation_is_idempotent: bool,
) -> bool:
    safe_method = method.upper() in {"GET", "HEAD"}
    return (
        status_code in TRANSIENT_STATUS_CODES
        and (safe_method or operation_is_idempotent)
    )
```
Stosuj ograniczone liczbą próby z rosnącymi opóźnieniami i jitterem (losowym rozrzutem). W telemetrii zachowuj trace ID usługi i identyfikator sesji aplikacji, ale **nie loguj tokenów i nie ujawniaj identyfikatorów sesji użytkownikom końcowym**.

## D5. Zwalnianie zakończonych sesji
_Źródło: [unit 5] + [docs: https://learn.microsoft.com/azure/container-apps/sessions-code-interpreter]_

Cooldown sprząta nieaktywne sesje automatycznie, ale backend może usunąć sesję interpretera wcześniej: to zwalnia pojemność i usuwa dane tymczasowe, gdy przepływ się kończy lub użytkownik kończy rozmowę. **Nie usuwaj sesji, gdy inne żądanie tej rozmowy jeszcze trwa.** Usuwanie to uwierzytelnione `DELETE`; odpowiedź **`204 No Content`** potwierdza, że środowiska już nie ma.

```http
DELETE <POOL_MANAGEMENT_ENDPOINT>/session?api-version=2025-02-02-preview&identifier=<SESSION_ID>
Authorization: Bearer <ACCESS_TOKEN>
```
Jeśli usunięcie się nie powiedzie, zgłoś to w telemetrii zamiast twierdzić, że sprzątanie się udało; cooldown pozostaje zabezpieczeniem, ale pojemność lub wrażliwe dane tymczasowe mogą pozostać do jego upływu. [unit 5]

> Uwaga o dokumentacji: strona "Use session pools" w tabeli punktów końcowych interpretera nadal wymienia starsze ścieżki (`code/execute`, `files/upload`) i `api-version=2024-02-02-preview`, podczas gdy strona "code interpreter" (i moduł) używa `/executions` i `/files` z `2025-10-02-preview`; trzymaj się wersji ze strony code interpreter i sprawdzaj aktualne API. [docs: https://learn.microsoft.com/azure/container-apps/session-pool]

## D6. Niezawodne operowanie przepływami sesji
_Źródło: [unit 5]_

Sandbox sesji ogranicza wpływ kodu, a walidacja, autoryzacja, limity czasu i sprawdzanie wyniku w backendzie decydują, co aplikacja akceptuje; **żadna z warstw nie zastępuje drugiej**.

- **Waliduj przed wykonaniem:** rozmiar ładunku, język, żądaną operację i wymagane dane wejściowe.
- **Autoryzuj każdą operację:** uwierzytelniony użytkownik jest właścicielem rozmowy i powiązania sesji po stronie serwera.
- **Dane trwałe trzymaj gdzie indziej:** pliki sesji to robocze dane tymczasowe.
- **Klasyfikuj błędy:** transport, wykonanie, walidacja i wygaśnięcie sesji osobno.
- **Ponawiaj z dowodem:** tylko ograniczone, idempotentne operacje po sprawdzeniu stanu.
- **Sprzątaj świadomie:** usuwaj zakończone sesje, gdy liczy się natychmiastowe usunięcie danych lub odzyskanie pojemności.
- **Loguj bezpiecznie:** czasy, statusy i trace ID, bez tokenów, wrażliwego kodu i surowych dokumentów.

---

# CZĘŚĆ E - ĆWICZENIA

Przeczytaj części A-D. Każde ćwiczenie ma ten sam układ pól (przewodnik stylu §5). Ćwiczenia I1 i I2 tworzą płatną pulę sesji (rozliczaną za czas trwania sesji): rób je w jednej grupie zasobów i wykonaj **Sprzątanie** po I2 (I2 kontynuuje I1). Kod klienta i SDK jest w Pythonie; odpowiedniki C# są w ćwiczeniach z kodem.

> Wersje sprawdzone 2026-11-22 (źródło: pypi.org, api.nuget.org, `az version`, testy lokalne): Python 3.12; requests 2.34.2; azure-identity 1.27.0; azure-mgmt-appcontainers 5.1.0; C#: .NET SDK 10.0, Azure.Identity 1.21.0, Azure.ResourceManager.AppContainers 1.6.0 (pule sesji: `SessionPoolResource`); Azure CLI 2.90.0, rozszerzenie containerapp 1.3.0b6 (beta). Przed użyciem sprawdź PyPI i NuGet.

## I1. Pula interpretera, rola Session Executor i klient REST w Pythonie (lab modułu)
(rozwinięcie jednostki 6; scenariusz: [unit 6])

**Gdzie / Plik:** terminal z Azure CLI (Bash) w pustym katalogu; pliki `sessions_client.py` i `demo.py`.

> Uwaga o labie: lab Microsoftu dostarcza paczkę startową (`azdeploy.py`, aplikację Flask w `client/` z pustymi sekcjami BEGIN/END do uzupełnienia) i pokazuje wyniki w przeglądarce; tutaj budujesz samodzielny klient konsolowy, który robi te same kroki (upload, wykonanie, lista plików, pobranie, błąd wykonania, usunięcie). Treść ciała `executions` w labie jest **płaska** (`"codeInputType": "Inline"`, `"executionType": "Synchronous"`, `timeoutInSeconds`, `outputStreamsMaxLength`), a w jednostce 4 i dokumentacji jest w obiekcie `properties` z małymi literami (`"inline"`, `"synchronous"`). Poniższy klient trzyma się formatu z dokumentacji i jednostki 4 oraz czyta odpowiedź z `properties` lub z korzenia (oba układy) [unit 6] [unit 4] [docs: https://learn.microsoft.com/azure/container-apps/sessions-code-interpreter].

`sessions_client.py` (przetestowany lokalnie wobec serwera-atrapy odtwarzającego kontrakt z dokumentacji):
```python
from __future__ import annotations

import os
from dataclasses import dataclass
from pathlib import Path
from typing import Any
from urllib.parse import quote
from uuid import uuid4

import requests
from azure.identity import DefaultAzureCredential

API_VERSION = "2025-10-02-preview"          # executions i files
SESSION_API_VERSION = "2025-02-02-preview"  # get i delete session
TOKEN_SCOPE = "https://dynamicsessions.io/.default"


class SessionTransportError(RuntimeError):
    """Blad warstwy HTTP: autoryzacja, pojemnosc, siec, usluga."""


class CodeExecutionError(RuntimeError):
    """Usluga uruchomila kod, ale kod zakonczyl sie bledem."""


@dataclass
class DynamicSessionClient:
    endpoint: str                     # poolManagementEndpoint
    credential: Any = None
    identifier: str = ""

    def __post_init__(self) -> None:
        self.endpoint = self.endpoint.rstrip("/")
        self.credential = self.credential or DefaultAzureCredential()   # (1)
        self.identifier = self.identifier or str(uuid4())               # (2)

    def _headers(self) -> dict[str, str]:
        token = self.credential.get_token(TOKEN_SCOPE).token            # (3)
        return {"Authorization": f"Bearer {token}"}

    def _request(self, method: str, path: str, *, version: str = API_VERSION,
                 timeout: tuple[int, int] = (5, 30), **kwargs: Any) -> requests.Response:
        params = {"api-version": version, "identifier": self.identifier}  # (4)
        try:
            response = requests.request(method, f"{self.endpoint}{path}",
                                        headers=self._headers(), params=params,
                                        timeout=timeout, **kwargs)
            response.raise_for_status()
        except requests.Timeout as error:
            raise SessionTransportError("timeout: nie wiadomo, czy kod sie wykonal") from error
        except requests.RequestException as error:
            raise SessionTransportError(str(error)) from error
        return response

    def upload_file(self, path: Path) -> dict[str, Any]:
        with path.open("rb") as source:                                  # (5)
            response = self._request("POST", "/files",
                                     files={"file": (path.name, source, "application/octet-stream")})
        return response.json()

    def execute_code(self, code: str) -> dict[str, Any]:
        response = self._request("POST", "/executions", timeout=(5, 90), json={
            "properties": {"codeInputType": "inline", "executionType": "synchronous",
                           "code": code, "timeoutInSeconds": 60}})      # (6)
        body = response.json()
        data = body.get("properties", body)                              # (7)
        if data.get("status") != "Succeeded":                            # (8)
            result = data.get("result") or {}
            raise CodeExecutionError(str(result.get("stderr") or data.get("error") or "execution failed"))
        return data

    def list_files(self) -> list[dict[str, Any]]:
        return self._request("GET", "/files", timeout=(5, 15)).json().get("value", [])

    def download_file(self, name: str) -> bytes:
        if Path(name).name != name:                                      # (9)
            raise ValueError("nazwa pliku bez katalogu")
        return self._request("GET", f"/files/{quote(name, safe='')}/content").content

    def get_session(self) -> dict[str, Any]:
        return self._request("GET", "/session", version=SESSION_API_VERSION).json()

    def delete_session(self) -> int:
        return self._request("DELETE", "/session", version=SESSION_API_VERSION, timeout=(5, 15)).status_code
```
Objaśnienie: (1) `DefaultAzureCredential`: lokalnie bierze Twoje logowanie z `az login`, w Azure tożsamość zarządzaną; (2) identyfikator tworzy **backend** (UUID), nie klient; (3) token dla odbiorcy `https://dynamicsessions.io` przy każdej operacji; (4) każde żądanie niesie `api-version` i ten sam `identifier`, więc trafia do tej samej sesji; (5) plik zamykany także przy błędzie; (6) ciało zgodne z dokumentacją, `timeoutInSeconds` 60 (limit platformy to 220); (7) odpowiedź może być w `properties` lub w korzeniu: schemat odpowiedzi nie jest opisany w jednostce, więc obsługujemy oba układy [autor: niezweryfikowane]; (8) **HTTP 200 nie wystarcza**: sprawdzamy `status` wykonania, a błąd kodu zgłaszamy jako `CodeExecutionError` (inny niż błąd transportu); (9) odrzucenie nazwy z katalogiem (path traversal). Wersje API jak w D3 i D5.

`demo.py`:
```python
import os
from pathlib import Path

from sessions_client import CodeExecutionError, DynamicSessionClient

ANALYSIS = """
import csv, json
rows = list(csv.DictReader(open('/mnt/data/operational-data.csv')))
total = sum(int(r['requests']) for r in rows)
summary = {'months': len(rows), 'total': total, 'average': total / len(rows)}
json.dump(summary, open('/mnt/data/summary.json', 'w'))
print(json.dumps(summary))
"""

Path("operational-data.csv").write_text(
    "month,requests\nJan,100\nFeb,200\nMar,300\nApr,400\n", encoding="utf-8")

client = DynamicSessionClient(os.environ["SESSION_POOL_ENDPOINT"])        # identyfikator tworzy backend
print("1 upload      :", client.upload_file(Path("operational-data.csv")))
print("2 execute     :", client.execute_code(ANALYSIS)["result"])
print("3 files       :", [f["properties"]["filename"] for f in client.list_files()])
print("4 download    :", client.download_file("summary.json").decode())
try:
    client.execute_code("raise RuntimeError('celowy blad')")
except CodeExecutionError as error:
    print("5 execution error (HTTP OK, kod padl):", error)
print("6 get session :", client.get_session())
print("7 delete      :", client.delete_session())
```
Skrypt analizy (czyta CSV z `/mnt/data`) sprawdzono lokalnie: wynik `{"months": 4, "total": 1000, "average": 250.0}`, czyli cztery miesiące, 1000 żądań łącznie i średnia 250, tak jak w labie [unit 6] [autor: test 2026-11-22].

Polecenia:
```bash
RG=rg-aca-sessions-i1; LOC=westus2; POOL=ai-document-sessions
az extension add --name containerapp --upgrade --allow-preview true --yes
az provider register --namespace Microsoft.App
az group create --name $RG --location $LOC
az containerapp sessionpool create --name $POOL --resource-group $RG --location $LOC \
    --container-type PythonLTS --max-sessions 5 --cooldown-period 300 --network-status EgressDisabled   # (1)
POOL_ID=$(az containerapp sessionpool show -n $POOL -g $RG --query id -o tsv)
export SESSION_POOL_ENDPOINT=$(az containerapp sessionpool show -n $POOL -g $RG \
    --query "properties.poolManagementEndpoint" -o tsv)                                                  # (2)
ME=$(az ad signed-in-user show --query id -o tsv)
az role assignment create --role "Azure ContainerApps Session Executor" --assignee $ME --scope $POOL_ID  # (3)
pip install requests==2.34.2 azure-identity==1.27.0
python demo.py                                                                                           # (4)
```
Objaśnienie: (1) pula interpretera Pythona z pięcioma sesjami, cooldownem 300 s (minimum dopuszczalne) i wyłączonym egress (jak w labie: pięć sesji, 300 s, egress wyłączony [unit 6]); (2) punkt końcowy zarządzania; (3) rola na zakresie puli dla Twojego konta (propagacja ról może potrwać kilka minut); (4) klient wykonuje kroki 1-7. Region musi obsługiwać sesje dynamiczne (przykład z dokumentacji używa `westus2`); w razie błędu wdrożenia zmień region [unit 6] [docs: https://learn.microsoft.com/azure/container-apps/session-pool].

Typowe problemy z labu [unit 6]: błąd autoryzacji tuż po przypisaniu roli - odczekaj kilka minut; potwierdź, że rola Session Executor jest na zakresie puli; po długiej przerwie `operational-data.csv` znika z sesji (cooldown 300 s), więc wgraj plik ponownie; nowa konsola wymaga ponownego załadowania zmiennych środowiskowych.

**Odpowiednik C# (.NET)**

Różnice: zamiast `requests` używasz `HttpClient` i `MultipartFormDataContent` (plik) oraz `JsonContent` (ciało JSON), a token bierzesz z `DefaultAzureCredential.GetTokenAsync(new TokenRequestContext(...))`. Błąd HTTP zamieniasz na wyjątek metodą `EnsureSuccessStatusCode()`, a **błąd wykonania kodu** sprawdzasz osobno po polu `status` (to samo rozróżnienie co w Pythonie). Projekt: `dotnet new console -f net10.0`, `dotnet add package Azure.Identity --version 1.21.0`; zmienna `SESSION_POOL_ENDPOINT` i plik `operational-data.csv` jak wyżej.
```csharp
using System.Net.Http.Headers;
using System.Net.Http.Json;
using System.Text.Json;
using Azure.Core;
using Azure.Identity;

const string ApiVersion = "2025-10-02-preview";          // executions i files
const string SessionApiVersion = "2025-02-02-preview";   // get i delete session
string endpoint = Environment.GetEnvironmentVariable("SESSION_POOL_ENDPOINT")!.TrimEnd('/');
string identifier = Guid.NewGuid().ToString();            // identyfikator tworzy backend

var credential = new DefaultAzureCredential();
using var http = new HttpClient { Timeout = TimeSpan.FromSeconds(90) };

async Task<HttpResponseMessage> SendAsync(HttpMethod method, string path, string version, HttpContent? content = null)
{
    AccessToken token = await credential.GetTokenAsync(
        new TokenRequestContext(new[] { "https://dynamicsessions.io/.default" }));
    var request = new HttpRequestMessage(method, $"{endpoint}{path}?api-version={version}&identifier={identifier}")
    {
        Content = content
    };
    request.Headers.Authorization = new AuthenticationHeaderValue("Bearer", token.Token);
    var response = await http.SendAsync(request);
    response.EnsureSuccessStatusCode();                    // blad transportu = wyjatek
    return response;
}

async Task<JsonElement> ExecuteAsync(string code)
{
    var body = new { properties = new { codeInputType = "inline", executionType = "synchronous", code, timeoutInSeconds = 60 } };
    using var response = await SendAsync(HttpMethod.Post, "/executions", ApiVersion, JsonContent.Create(body));
    JsonElement root = (await response.Content.ReadFromJsonAsync<JsonElement>());
    JsonElement data = root.TryGetProperty("properties", out var p) ? p : root;
    if (data.GetProperty("status").GetString() != "Succeeded")   // HTTP OK, ale kod padl
        throw new InvalidOperationException("execution failed: " + data.GetRawText());
    return data;
}

// 1. upload
await using (var stream = File.OpenRead("operational-data.csv"))
using (var form = new MultipartFormDataContent())
{
    form.Add(new StreamContent(stream), "file", "operational-data.csv");
    using var _ = await SendAsync(HttpMethod.Post, "/files", ApiVersion, form);
}

// 2. execute
var result = await ExecuteAsync("print(sum(range(5)))");
Console.WriteLine(result.GetProperty("result").GetRawText());

// 3. list files
using (var list = await SendAsync(HttpMethod.Get, "/files", ApiVersion))
    Console.WriteLine(await list.Content.ReadAsStringAsync());

// 4. delete
using (var del = await SendAsync(HttpMethod.Delete, "/session", SessionApiVersion))
    Console.WriteLine((int)del.StatusCode);
```
[docs: https://learn.microsoft.com/azure/container-apps/sessions-code-interpreter] [autor: test 2026-11-22] (kod skompilowano na .NET SDK 10.0.401 z Azure.Identity 1.21.0; nie uruchamiano go na prawdziwej puli sesji; krok pobrania pliku i lista plików są analogiczne do Pythona).

**Koszt:** sesje interpretera są rozliczane według **czasu trwania każdej sesji** [docs: https://learn.microsoft.com/azure/container-apps/sessions-code-interpreter]; stawki i ewentualne darmowe progi: Niesprawdzone - zweryfikuj w https://learn.microsoft.com/azure/container-apps/billing. Ćwiczenie tworzy kilka krótkich sesji (pula nic nie kosztuje, dopóki nie przydzielasz sesji, o ile nie ma gotowych instancji; to założenie sprawdź w cenniku [autor: niezweryfikowane]). Wykonaj Sprzątanie.

**Zadania:** (a) uruchom `python demo.py` i potwierdź kroki 1-7; (b) **zepsuj to - limit czasu:** wykonaj kod `import time; time.sleep(120)` z `timeoutInSeconds` 60 (zmień wartość w `execute_code`) i zobacz, jak klient to klasyfikuje; (c) **zepsuj to - wygaśnięcie sesji:** po kroku 1 odczekaj ponad 300 s bez żądań, potem wylistuj pliki tym samym identyfikatorem (w osobnym skrypcie albo przez pozostawienie klienta w `python -i`); (d) **zepsuj to - zła rola:** w osobnym koncie (bez roli Session Executor na puli) wykonaj upload i przeczytaj błąd.

**Sprawdź:** (a) kroki wypisują: odpowiedź uploadu, wynik z `stdout` równy `{"months": 4, "total": 1000, "average": 250.0}`, listę plików `operational-data.csv` i `summary.json`, treść `summary.json`, komunikat `RuntimeError`/"celowy blad", metadane sesji i `204` [unit 6]; (b) wykonanie nie kończy się statusem `Succeeded`, więc klient zgłasza `CodeExecutionError` albo timeout transportu (dokładny status zależy od usługi) [autor: niezweryfikowane]; (c) po wygaśnięciu sesji lista plików jest pusta, bo identyfikator wskazuje już nową, pustą sesję [unit 5]; (d) błąd HTTP 401/403 (brak uprawnień), zgłoszony jako `SessionTransportError` [autor: niezweryfikowane].

**Sprzątanie:** pula jest potrzebna jeszcze w I2 i I3, więc usuń ją po nich (Sprzątanie z I2). Jeśli kończysz na I1: `az group delete --name $RG --yes --no-wait`, a potem `az group exists --name $RG` (po kilku minutach powinno wypisać `false`).

## I2. Egress i cooldown: eksperymenty z konfiguracją puli
(rozwinięcie sekcji B2-B3, D1; kontynuacja I1: ta sama grupa `rg-aca-sessions-i1`, pula `ai-document-sessions`, plik `sessions_client.py` i zmienne z I1)

**Gdzie / Plik:** terminal z Azure CLI (Bash) i Python w katalogu z I1; plik `egress_test.py`.

`egress_test.py`:
```python
import os

from sessions_client import CodeExecutionError, DynamicSessionClient

client = DynamicSessionClient(os.environ["SESSION_POOL_ENDPOINT"])
CODE = "import urllib.request; print(urllib.request.urlopen('https://example.com', timeout=10).status)"
try:
    print("wynik:", client.execute_code(CODE)["result"])
except CodeExecutionError as error:
    print("kod padl:", error)
finally:
    client.delete_session()                 # zwolnij pojemnosc
```
```bash
python egress_test.py                                                                            # (1) egress wylaczony

az containerapp sessionpool update -n $POOL -g $RG --network-status EgressEnabled                # (2)
python egress_test.py                                                                            # (3)

az containerapp sessionpool update -n $POOL -g $RG --network-status EgressDisabled               # (4)
az containerapp sessionpool show -n $POOL -g $RG \
    --query "{max:properties.scaleConfiguration.maxConcurrentSessions, egress:properties.sessionNetworkConfiguration.status}" -o table   # (5)
```
Objaśnienie: (1) przy wyłączonym egress żądanie sieciowe z kodu nie może się powieść; (2) włączenie egress zmienia ustawienie **całej puli**; (3) ten sam kod powinien teraz dostać odpowiedź; (4) **przywróć bezpieczne ustawienie**; (5) odczyt ustawień puli (nazwy pól jak w modelu `SessionPoolProperties`).

**Koszt:** jak w I1 (te same zasoby; kilka krótkich sesji).

**Zadania:** (a) uruchom test przy wyłączonym egress i przeczytaj błąd; (b) włącz egress i powtórz; (c) **zepsuj to:** przywróć `EgressDisabled` i potwierdź, że kod znowu nie ma dostępu do sieci; (d) zmień `--max-sessions` na 1, uruchom dwa klienty równolegle (dwa terminale z `python demo.py`) i obserwuj, co się dzieje, gdy limit współbieżnych sesji zostanie przekroczony; potem przywróć 5.

**Sprawdź:** (a) kod kończy się błędem sieci (wykonanie ze statusem innym niż `Succeeded` i komunikatem o braku połączenia) [unit 3]; (b) po włączeniu egress odpowiedź to `200` [unit 3]; (c) błąd wraca [unit 3]; (d) drugi klient czeka lub dostaje błąd pojemności (zachowanie przy osiągnięciu limitu: sprawdź i zapisz obserwację) [unit 3] [autor: niezweryfikowane].

**Sprzątanie:**
```bash
az group delete --name $RG --yes --no-wait
az group exists --name $RG      # po kilku minutach powinno wypisać false
```
Usuwa pulę z I1 i I2. Wykonaj dopiero po I3.

## I3. Odczyt konfiguracji pul przez SDK zarządzania: Python i odpowiednik C# (.NET)
(rozwinięcie sekcji B1-B3; wymaga puli z I1 przed jej Sprzątaniem w I2)

**Gdzie / Plik:** `list_pools.py` (Python) i `Program.cs` (C#); `az login`.

```bash
pip install azure-mgmt-appcontainers==5.1.0 azure-identity==1.27.0
export AZURE_SUBSCRIPTION_ID=$(az account show --query id -o tsv)
export RESOURCE_GROUP=rg-aca-sessions-i1
```
`list_pools.py`:
```python
import os

from azure.identity import DefaultAzureCredential
from azure.mgmt.appcontainers import ContainerAppsAPIClient

client = ContainerAppsAPIClient(DefaultAzureCredential(), os.environ["AZURE_SUBSCRIPTION_ID"])  # (1)

for pool in client.container_apps_session_pools.list_by_resource_group(os.environ["RESOURCE_GROUP"]):  # (2)
    p = pool.properties
    print(pool.name, p.container_type, p.provisioning_state)
    print("  endpoint :", p.pool_management_endpoint)                                  # (3)
    print("  max      :", p.scale_configuration.max_concurrent_sessions)                # (4)
    lifecycle = p.dynamic_pool_configuration.lifecycle_configuration
    print("  cooldown :", lifecycle.cooldown_period_in_seconds, lifecycle.lifecycle_type)
    print("  egress   :", p.session_network_configuration.status)
```
Objaśnienie: (1) ten sam klient zarządzania co w modułach 04-06; (2) lista pul w grupie zasobów (`container_apps_session_pools`); (3) punkt końcowy zarządzania do wywołań REST; (4) pojemność, cooldown i status egress, czyli ustawienia z części B. Skrypt skompilowano (`py_compile`), a nazwy pól sprawdzono w modelach `SessionPoolProperties`, `ScaleConfiguration`, `LifecycleConfiguration` i `SessionNetworkConfiguration` w pakiecie 5.1.0; nie uruchamiano go na prawdziwej subskrypcji [autor: test 2026-11-22].

**Odpowiednik C# (.NET)**

Różnice: pule sesji pobierasz z **zasobu grupy zasobów** (`ResourceGroupResource.GetSessionPools()`), typ zasobu to `SessionPoolResource` (nie "ContainerApp...", jak w innych zasobach tego pakietu), a właściwości nazywają się inaczej niż w Pythonie: `PoolManagementEndpoint`, `ScaleConfiguration.MaxConcurrentSessions`, `DynamicPoolLifecycleConfiguration.CooldownPeriodInSeconds`, `SessionNetworkStatus`. Projekt: `dotnet add package Azure.ResourceManager.AppContainers --version 1.6.0` i `Azure.Identity --version 1.21.0`.
```csharp
using Azure.Identity;
using Azure.ResourceManager;
using Azure.ResourceManager.AppContainers;
using Azure.ResourceManager.Resources;

string sub = Environment.GetEnvironmentVariable("AZURE_SUBSCRIPTION_ID")!;
string rg = Environment.GetEnvironmentVariable("RESOURCE_GROUP")!;

var arm = new ArmClient(new DefaultAzureCredential());
ResourceGroupResource group = arm.GetResourceGroupResource(ResourceGroupResource.CreateResourceIdentifier(sub, rg));

await foreach (SessionPoolResource pool in group.GetSessionPools().GetAllAsync())
{
    var d = pool.Data;
    Console.WriteLine($"{d.Name} {d.ContainerType} {d.ProvisioningState}");
    Console.WriteLine($"  endpoint : {d.PoolManagementEndpoint}");
    Console.WriteLine($"  max      : {d.ScaleConfiguration?.MaxConcurrentSessions}");
    Console.WriteLine($"  cooldown : {d.DynamicPoolLifecycleConfiguration?.CooldownPeriodInSeconds} {d.DynamicPoolLifecycleConfiguration?.LifecycleType}");
    Console.WriteLine($"  egress   : {d.SessionNetworkStatus}");
}
```
[docs: https://learn.microsoft.com/dotnet/api/overview/azure/resourcemanager.appcontainers-readme?view=azure-dotnet] [autor: test 2026-11-22] (kod skompilowano na .NET SDK 10.0.401; nie uruchamiano go na prawdziwej subskrypcji).

**Koszt:** brak kosztu własnego (odczyt metadanych); koszt puli jak w I1.

**Zadania:** (a) uruchom `python list_pools.py` i porównaj wynik z `az containerapp sessionpool show`; (b) **zepsuj to:** ustaw `RESOURCE_GROUP=nie-ma-takiej` i przeczytaj błąd; (c) uruchom wersję C# i porównaj pola.

**Sprawdź:** (a) te same wartości: typ `PythonLTS`, maksimum 5 (lub 1, jeśli nie przywróciłeś), cooldown 300, egress `EgressDisabled`; (b) błąd "grupa zasobów nie istnieje" (ResourceGroupNotFound); dokładny komunikat zależy od wersji SDK [autor: niezweryfikowane]; (c) te same wartości w formacie C#.

**Sprzątanie:** brak (odczyt metadanych); po I3 wykonaj Sprzątanie z I2.

---

# CZĘŚĆ G - QUIZ Z MODUŁU MICROSOFTU (5 pytań) + wyjaśnienia

Learn nie pokazuje poprawnych odpowiedzi w sprawdzianie wiedzy, więc odpowiedzi są wnioskowane z treści jednostek; przy każdym pytaniu jest krótki znacznik.

## G1. Zastrzeżony konwerter z natywnymi binariami - jaka pula?
-> *Pula z własnym kontenerem (custom container) z konwerterem w obrazie.* Wbudowany interpreter (przez REST lub `SessionsPythonREPLTool`) nie obsługuje natywnych programów ani własnych bibliotek. *(odpowiedź wnioskowana: unit 2, sekcje "Choose the built-in code interpreter" i "Choose a custom container")*

## G2. Niezaufany Python na wgranych plikach - jak ograniczyć ryzyko wycieku danych?
-> *Ustawić `--network-status EgressDisabled` na puli.* `EgressEnabled` z ograniczeniem przez tożsamość backendu nie blokuje ruchu z sesji; dłuższy cooldown tylko wydłuża retencję plików. *(odpowiedź wnioskowana: unit 3, sekcja "Control outbound network access")*

## G3. Wielodostępny backend - jak zarządzać identyfikatorami sesji?
-> *Generować nieprzewidywalny identyfikator po stronie serwera i przechowywać jego powiązanie z użytkownikiem lub rozmową w stanie backendu.* E-mail jako identyfikator jest przewidywalny, a akceptowanie identyfikatora z klienta pozwala adresować cudzą sesję. *(odpowiedź wnioskowana: unit 4, sekcja "Create secure session identifiers")*

## G4. API zwraca sukces HTTP, ale Python zgłosił wyjątek - jak sklasyfikować?
-> *Jako execution failure (błąd wykonania).* To nie jest błąd transportu (HTTP się udał) ani sukces. *(odpowiedź wnioskowana: unit 5, sekcja "Separate failure categories")*

## G5. Timeout po stronie backendu, nie wiadomo, czy kod miał skutki uboczne - najbezpieczniejsza polityka
-> *Ponawiać tylko wtedy, gdy operacja jest idempotentna, a backend potrafi ocenić wynikowy stan.* Natychmiastowe ponawianie każdego timeoutu z tym samym identyfikatorem może zdublować skutki, a nowy identyfikator nie rozwiązuje niejednoznaczności. *(odpowiedź wnioskowana: unit 5, sekcja "Apply retries safely")*

---

# CZĘŚĆ H - PYTANIA KONTROLNE (odpowiedz bez zaglądania)

## H1. Co daje pula sesji i jaka jest granica izolacji?
<details><summary>Odpowiedź</summary>

Wstępnie rozgrzane środowiska przydzielane w milisekundach; każda sesja działa w osobnej piaskownicy izolowanej przez Hyper-V (procesy i system plików oddzielone od hosta i innych sesji). Sesja jest efemeryczna.
</details>

## H2. Kiedy wbudowany interpreter, a kiedy własny kontener?
<details><summary>Odpowiedź</summary>

Interpreter: analiza CSV, obliczenia, wizualizacje, krótkie skrypty agenta, gdy wystarczy obsługiwany runtime. Własny kontener: specjalne pakiety systemowe, binaria natywne, język spoza interpretera, własny protokół (wymaga środowiska z profilami obciążeń; odpowiadasz za obraz, ingress, logi i sondy).
</details>

## H3. Wymień limity i wartości puli, które warto znać.
<details><summary>Odpowiedź</summary>

`--max-sessions` maks. 600; `--cooldown-period` 300-3600 s (domyślnie 300); `--network-status` domyślnie `EgressDisabled`; typy `PythonLTS` (domyślny), `NodeLTS`, `Shell`, `CustomContainer`; wykonanie maks. 220 s; wgrywanie pliku do 128 MB.
</details>

## H4. Jakiej roli i jakiego odbiorcy tokenu wymaga wywołanie API zarządzania puli?
<details><summary>Odpowiedź</summary>

Roli Azure ContainerApps Session Executor (dokumentacja: także Contributor) na puli; token Entra z odbiorcą (audience) `https://dynamicsessions.io` (zakres `https://dynamicsessions.io/.default`).
</details>

## H5. Jak bezpiecznie obsługiwać identyfikatory sesji?
<details><summary>Odpowiedź</summary>

Generować je po stronie serwera (nieprzewidywalnie, np. UUID) dla uwierzytelnionego użytkownika lub rozmowy, trzymać powiązanie w stanie backendu, nie przyjmować identyfikatora od klienta i nie ujawniać go użytkownikom; sam identyfikator nie uwierzytelnia.
</details>

## H6. Gdzie lądują wgrane pliki i jak wygląda ścieżka pobrania?
<details><summary>Odpowiedź</summary>

W `/mnt/data` sesji; wgranie: `POST /files` (multipart); lista: `GET /files`; pobranie: `GET /files/{nazwa}/content` (nazwa kodowana jako jeden segment URL i walidowana przed zapisem lokalnym).
</details>

## H7. Jak odróżnić transport failure od execution failure?
<details><summary>Odpowiedź</summary>

Transport failure: żądanie API nie zostało zrealizowane (uwierzytelnianie, autoryzacja, pojemność, sieć, usługa). Execution failure: HTTP się udał, ale kod w sesji zgłosił wyjątek lub zwrócił nieudany status wykonania. Dodatkowo validation failure: wynik niezgodny z kontraktem aplikacji.
</details>

## H8. Kiedy wolno ponawiać żądanie wykonania kodu?
<details><summary>Odpowiedź</summary>

Tylko dla operacji idempotentnych, gdy backend umie zweryfikować wynik; ograniczona liczba prób, rosnące opóźnienia z jitterem; timeout jest niejednoznaczny (wykonanie mogło się zakończyć). Odczyty (GET/HEAD) i zapytania o status są naturalnymi kandydatami.
</details>

## H9. Co zwróci udane usunięcie sesji i po co je robić?
<details><summary>Odpowiedź</summary>

`DELETE .../session` odpowiada `204 No Content`. Zwalnia pojemność i natychmiast usuwa dane tymczasowe (zamiast czekać na cooldown); nie usuwaj sesji, gdy trwa inne żądanie tej rozmowy.
</details>

---

# CZĘŚĆ J - CO DALEJ

## J1. Następny krok i pokrycie egzaminu
_Źródło: [docs: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/ai-200]_

To ostatni moduł ścieżki "Deploy and manage apps on Azure Container Apps" (moduły 04-07). Dalej w kursie AI-200T00: moduły o Kubernetes (AKS) - wdrażanie, konfiguracja i monitorowanie - oraz moduły o bazach danych (Cosmos DB, PostgreSQL, Redis). Wróć do części B i D, gdy projektujesz bezpieczną pulę i obsługę błędów.

**Pokrycie egzaminu** (wiersze z `sources/_exam/skill-map.md`, stan study guide z 2026-10-30; dla tego modułu nie ma odpowiadającej umiejętności):

| Umiejętność (study guide) | Domena | Status | Gdzie |
|---|---|---|---|
| (brak osobnej umiejętności o sesjach dynamicznych w study guide) | 1 | tu tylko wzmianka (wiedza pokrewna; moduł jest w kursie, ale nie ma wiersza w skill-map) | `07-dynamic-sessions-azure-container-apps.md` |
| Deploy applications to Azure Container Apps, including environment configuration and revision management | 1 | uczone w modułach 04 i 05 | `04-deploy-containers-azure-container-apps.md`; `05-manage-containers-azure-container-apps.md` |
| Implement event-driven scaling by using KEDA in Container Apps | 1 | uczone w module 06 | `06-scale-containers-azure-container-apps.md` |

## J2. Uczciwie o egzaminie
_Źródło: [autor]_

Ten moduł nie odpowiada żadnej wprost wymienionej umiejętności study guide AI-200 (stan 2026-10-30), więc na egzaminie spodziewaj się raczej pytań z modułów 04-06. Wiedza z modułu 07 jest wartościowa jako pokrewna (bezpieczne wykonywanie kodu AI, tożsamości, identyfikatory, klasyfikacja błędów). Quiz jest krótki (5 pytań); jeśli pojawią się pytania o sesje, ćwicz: wybór puli, egress wyłączony domyślnie, identyfikatory po stronie serwera, rola Session Executor i odbiorca `https://dynamicsessions.io`, rozróżnienie błędów transportu i wykonania oraz ponawianie tylko idempotentnych operacji.
