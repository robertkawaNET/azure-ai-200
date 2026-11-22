# Moduł 05: Zarządzanie kontenerami w Azure Container Apps (rewizje, cykl życia, logi, sondy zdrowia, zasoby)

Moduł Microsoft Learn: **Manage containers in Azure Container Apps** - https://learn.microsoft.com/en-us/training/modules/manage-containers-azure-container-apps/
Cel: po tym module umiesz bezpiecznie aktualizować obraz i zarządzać rewizjami, uruchamiać, zatrzymywać i restartować aplikację, diagnozować nieudaną rewizję po logach, skonfigurować sondy zdrowia (liveness, readiness) oraz dobrać CPU i pamięć.
Czas: ok. 3-4 h, orientacyjnie (moduł Learn ma ok. 68 min wg katalogu, w tym ćwiczenie ok. 30 min; tutaj dochodzą eksperymenty i ćwiczenie z SDK). Wymagania: Azure CLI, Python 3.12, ćwiczenie I1 korzysta z plików aplikacji z ćwiczenia I1 modułu 04 (`04-deploy-containers-azure-container-apps.md`). Koszt: ćwiczenia I1 i I2 tworzą płatne zasoby w chmurze (ACR, środowisko Container Apps, Log Analytics) - szczegóły w polach **Koszt**.

Źródła: `[unit N]` = jednostka modułu Learn, `[docs: URL]` = dokumentacja produktu, `[autor]` = dodane przez autora; pełna legenda: przewodnik stylu §3. "Moduł N" znaczy plik tutoriala `0N-*.md`.

---

# CZĘŚĆ A - AKTUALIZACJA OBRAZÓW I REWIZJE

## A1. Tagi i digesty: kiedy które
_Źródło: [unit 2]_

**Tag** obrazu (np. `dev`, `v2`) to wygodna, ale **zmienna** etykieta: ten sam tag może z czasem wskazywać inny obraz. **Digest** (skrót `sha256:...`) to **niezmienny** identyfikator konkretnej zawartości obrazu. Dla AI ważna jest identyfikowalność: chcesz powiązać zachowanie usługi z konkretnym buildem modelu. Gdy odwołujesz się do digestu, możesz udowodnić, jaki artefakt obsługiwał żądania podczas incydentu. Tag traktuj jako wygodę, nie gwarancję niezmienności.

(egzamin je lubi: produkcja = digest, np. `myregistry.azurecr.io/myapp@sha256:<digest>`; tag `latest` nie daje identyfikowalności)

## A2. Aktualizacja obrazu tworzy nową rewizję
_Źródło: [unit 2]_

Zmiana obrazu tworzy nową rewizję z nowym szablonem. Zanim skierujesz na nią ruch, sprawdzasz jej zdrowie; dzięki temu ruch produkcyjny nie trafia do rewizji, która nie startuje, nie przechodzi sond albo pada pod obciążeniem.

```azurecli
az containerapp update \
  --name <app-name> \
  --resource-group <resource-group> \
  --image <registry>/<repo>@sha256:<digest>      # (1)

az containerapp show \
  --name <app-name> \
  --resource-group <resource-group> \
  --query properties.template.containers          # (2)
```
Objaśnienie: (1) aktualizacja obrazu po digeście; (2) odczyt szablonu kontenerów, żeby potwierdzić, jaki obraz zapisała platforma.

Przypomnienie z modułu 04 (szczegóły: `04-deploy-containers-azure-container-apps.md` §B4): nową rewizję tworzą tylko zmiany w sekcji `template` (revision-scope), a nie zmiany w `configuration` (np. ingress, wartości sekretów). [docs: https://learn.microsoft.com/azure/container-apps/revisions]

## A3. Tryby rewizji i znaczenie słowa "active"
_Źródło: [unit 2]_

**Tryb rewizji** (revision mode) określa, ile rewizji może być aktywnych naraz:

| Tryb | Co robi | Kiedy |
|---|---|---|
| **Single** | tylko jedna aktywna rewizja | prostsze operacje; brak zaawansowanych rolloutów |
| **Multiple** | wiele aktywnych rewizji naraz | canary (wdrożenie na małej części ruchu) i podział ruchu (traffic splitting), np. test nowej wersji modelu |

**Active** to stan operacyjny: rewizja może przyjmować ruch. Rewizja może istnieć, ale nie być aktywna; nieaktywne warto trzymać do badania awarii i planowania wycofania (rollback). Podczas awarii często najpierw **dezaktywujesz** problematyczną rewizję, żeby nie dostawała ruchu.

Wartości parametru CLI `--revisions-mode`: `labels`, `multiple`, `single`. Zmiana trybu rewizji to zmiana application-scope. Podział ruchu i etykiety omawia moduł 06. [docs: https://learn.microsoft.com/cli/azure/containerapp?view=azure-cli-latest] [docs: https://learn.microsoft.com/azure/container-apps/revisions]

## A4. Lista, szczegóły, dezaktywacja i porządki
_Źródło: [unit 2]_

Rewizje to jednostka, którą badasz podczas rolloutu. Lista pokazuje, która jest aktywna; `show` pozwala porównać konfigurację działającej rewizji z uszkodzoną.

```azurecli
az containerapp revision list --name <app-name> --resource-group <resource-group> -o table

az containerapp revision show --name <app-name> --resource-group <resource-group> \
  --revision <revision-name>

az containerapp revision deactivate --name <app-name> --resource-group <resource-group> \
  --revision <revision-name>
```

**Dezaktywacja jest bezpieczniejszym pierwszym krokiem niż usunięcie**, bo zachowuje dowody i opcje wycofania. Gdy incydent minie i stare rewizje nie są już potrzebne, usuwasz je w ramach higieny (`az containerapp revision delete`). Wynika z tego odróżnienie dla egzaminu: `deactivate` zdejmuje rewizję z ruchu bez kasowania; `delete` kasuje; `az containerapp stop` zatrzymuje całą aplikację, a to zwykle zbyt szeroki ruch, gdy zły jest jeden release.

Dodatkowo CLI ma `az containerapp revision activate` (ponowna aktywacja), `az containerapp revision restart --revision <nazwa>` (restart pojedynczej rewizji, np. by pobrała nową wartość sekretu) i `az containerapp revision set-mode`. [docs: https://learn.microsoft.com/azure/container-apps/revisions-manage] [docs: https://learn.microsoft.com/azure/container-apps/revisions]

## A5. Dobre praktyki bezpiecznego rolloutu i retencja
_Źródło: [unit 2]_

- **Używaj digestów na produkcji:** niezmienne odwołanie do audytu po incydencie.
- **Sprawdź zdrowie rewizji przed przesunięciem ruchu:** status rewizji, sygnały sond i logi.
- **Dezaktywuj przed badaniem:** zachowuje dowody i zmniejsza ryzyko.
- **Zdefiniuj retencję:** Container Apps automatycznie czyści nieaktywne rewizje po przekroczeniu **100**; próg zmienisz parametrem `--max-inactive-revisions`. Dokumentacja potwierdza domyślne 100 nieaktywnych rewizji; w lokalnym teście (CLI 2.90.0, rozszerzenie 1.3.0b6) parametr jest oznaczony jako **Preview**. [docs: https://learn.microsoft.com/azure/container-apps/revisions] [autor: test 2026-11-22]

---

# CZĘŚĆ B - CYKL ŻYCIA APLIKACJI

## B1. Stop, restart czy skalowanie do zera
_Źródło: [unit 3]_

Zarządzanie cyklem życia wspiera kontrolę kosztów i reakcję na incydenty. Czasem trzeba wstrzymać usługę podczas awarii dostawcy, zrestartować po zmianie konfiguracji albo zawęzić skutki, badając złą rewizję.

| Akcja | Charakter | Kiedy |
|---|---|---|
| Skalowanie do zera (scale to zero) | wynika z konfiguracji skalowania | usługi sterowane zdarzeniami, ruch sporadyczny |
| **Stop** | jawna decyzja operacyjna | gwarancja, że aplikacja nie uruchomi replik na czas mitygacji |
| **Restart** | pomiędzy tymi dwoma | zmiana konfiguracji ma zadziałać od razu lub proces się zawiesił (deadlock, chwilowy problem zależności) |

Dla usług AI, które ładują duży model przy starcie, zbyt częsty restart zwiększa liczbę zimnych startów, więc restartuj z namysłem.

## B2. Start i stop aplikacji
_Źródło: [unit 3]_

Start i stop działają na **całą aplikację**, więc bywają zbyt szerokie, gdy problem dotyczy jednej rewizji; wtedy bezpieczniejsza jest dezaktywacja rewizji.

```azurecli
az containerapp stop  --name <app-name> --resource-group <resource-group>
az containerapp start --name <app-name> --resource-group <resource-group>
```

Przetestuj akcje cyklu życia w środowisku nieprodukcyjnym, zanim włączysz je do zautomatyzowanej reakcji na incydent.

## B3. Restart aplikacji po złym stanie uruchomieniowym
_Źródło: [unit 3]_

Restart wymusza restart replik i może wyczyścić chwilowe stany awarii. Nie zastępuje szukania przyczyny: połącz go z oglądaniem logów, żeby wiedzieć, czy naprawił problem, czy tylko odsunął następną awarię.

```azurecli
az containerapp restart --name <app-name> --resource-group <resource-group>
```

## B4. Powtarzalna lista kontrolna dla nieudanej rewizji
_Źródło: [unit 3]_

Najpierw potwierdź, **która** rewizja zawodzi i czy stan zdrowia raportowany przez platformę zgadza się z logami. Typowe wzorce awarii:

| Wzorzec | Objaw / przyczyna |
|---|---|
| **Image pull failures** (błędy pobierania obrazu) | brakujące lub błędne poświadczenia rejestru albo zła referencja obrazu |
| **Port mismatches** (niezgodne porty) | kontener nasłuchuje na innym porcie, niż oczekuje ingress |
| **Missing configuration** (brak konfiguracji) | brak lub błędna nazwa zmiennych środowiskowych albo sekretów |
| **Probe failures** (błędy sond) | sonda liveness lub readiness wskazuje złą ścieżkę albo nie daje czasu na załadowanie modelu |
| **Resource pressure** (brak zasobów) | kontener ginie przez limit pamięci lub zwalnia przez ograniczanie CPU (throttling) |

Zapytanie na początek, porównujące rewizje jednym rzutem oka:

```azurecli
az containerapp revision list \
  --name <app-name> \
  --resource-group <resource-group> \
  --query "[].{name:name,active:properties.active,health:properties.healthState}" \
  -o table
```

---

# CZĘŚĆ C - LOGI I ROZWIĄZYWANIE PROBLEMÓW

## C1. Strumieniowanie logów
_Źródło: [unit 4]_

Logi to główny sygnał diagnostyczny w zarządzanej platformie kontenerów: wiążą żądania z wersjami modelu, tłumaczą skoki opóźnień i pokazują błędy zależności bez dostępu do powłoki kontenera. Container Apps integruje się z Azure Monitor, a w trakcie incydentu możesz też oglądać logi na żywo (**log streaming**). Dostęp do logów zależy od konfiguracji środowiska: sprawdź, czy środowisko jest połączone z Log Analytics albo użyj strumieniowania.

```azurecli
az containerapp logs show \
  --name <app-name> \
  --resource-group <resource-group> \
  --follow
```

Strumień jest przydatny, gdy da się szybko odtworzyć błąd (żądanie z wyjątkiem, pętla awarii nowej rewizji). Łącz go z logowaniem strukturalnym, żeby filtrować po identyfikatorach żądania i rewizji.

## C2. Logi, które pomagają przy AI
_Źródło: [unit 4]_

Usługi AI zachowują się różnie dla różnych danych wejściowych, więc logi muszą mieć kontekst, ale bez wrażliwych treści: loguj **identyfikatory i metadane**, a nie surowe dokumenty czy prompty. Zalecane pola:

- **Request identifier:** identyfikator korelacji przekazywany między usługami.
- **Revision or build identifier:** wartość zgodna z tagiem lub digestem obrazu albo wersją wbudowaną w obraz.
- **Model version:** nazwa lub wersja wdrożenia modelu użytego w żądaniu.
- **Latency breakdown:** czas całkowity i, jeśli się da, czasy kluczowych zależności.

## C3. Procedura dla awarii związanej z jedną rewizją
_Źródło: [unit 4]_

Awarie rewizji dzielą się na: błędy startu, błędy sond, błędy konfiguracji i wyjątki w czasie działania. Aplikacja może np. zakończyć się od razu przez brak zmiennej środowiskowej albo wystartować, ale nie przejść readiness przez złą ścieżkę. Procedura, która szybko zawęża zakres:

1. Potwierdź, która rewizja jest aktywna, a która zawodzi.
2. Strumieniuj logi podczas odtwarzania problemu lub startu uszkodzonej rewizji.
3. Porównaj konfigurację działającej i uszkodzonej rewizji.
4. Zastosuj celową poprawkę i sprawdź, czy kolejna rewizja staje się gotowa.

## C4. Historia logów: Log Analytics i KQL
_Źródło: [unit 7] + [docs: https://learn.microsoft.com/azure/container-apps/log-monitoring]_

`az containerapp logs show` pokazuje tylko niedawne logi konsoli. Historię trzymasz w obszarze **Log Analytics** środowiska i odpytujesz językiem **KQL** (Kusto Query Language, język zapytań do logów). [unit 7]

Dokumentacja podaje trzy typy logów: **console** (stdout/stderr kontenerów), **system** (generowane przez usługę) i **HTTP** (warstwa ingress, po włączeniu diagnostic settings). Tabele (kategoria Custom logs): `ContainerAppConsoleLogs_CL` i `ContainerAppSystemLogs_CL`. W tabeli konsoli ważne kolumny to `ContainerAppName_s` (nazwa aplikacji), `RevisionName_s` (rewizja), `ContainerGroupName_g` (replika) i `Log_s` (komunikat). Nowe wpisy pojawiają się z opóźnieniem kilku minut. [docs: https://learn.microsoft.com/azure/container-apps/log-monitoring]

```azurecli
WORKSPACE_ID=$(az containerapp env show -n $ACA_ENVIRONMENT -g $RESOURCE_GROUP \
    --query properties.appLogsConfiguration.logAnalyticsConfiguration.customerId -o tsv)   # (1)

az monitor log-analytics query -w $WORKSPACE_ID \
    --analytics-query "ContainerAppConsoleLogs_CL | where ContainerAppName_s == '$CONTAINER_APP_NAME' | project TimeGenerated, Log_s | order by TimeGenerated desc | take 20" \
    -o table                                                                                  # (2)
```
Objaśnienie: (1) `customerId` obszaru Log Analytics wzięty ze środowiska; (2) 20 ostatnich komunikatów aplikacji. [unit 7] Wariant z błędami: dodaj `and Log_s contains 'error'`. Do zapytań potrzebne jest rozszerzenie `log-analytics` (`az extension add --name log-analytics`). [unit 7]

---

# CZĘŚĆ D - SONDY ZDROWIA (HEALTH PROBES)

## D1. Readiness kontra liveness (i startup)
_Źródło: [unit 5] + [docs: https://learn.microsoft.com/azure/container-apps/health-probes]_

**Sonda** (probe) to cykliczne sprawdzenie repliki. Sondy chronią użytkowników podczas rolloutów, bo nie wpuszczają ruchu do niezdrowych replik.

| Sonda | Pytanie | Skutek porażki |
|---|---|---|
| **Readiness** (gotowość) | "Czy ta replika może teraz przyjąć ruch?" | platforma przestaje kierować do niej żądania |
| **Liveness** (żywotność) | "Czy proces jest dość zdrowy, by działać dalej?" | platforma **restartuje** replikę |
| **Startup** (start, tylko dokumentacja) | "Czy aplikacja już się uruchomiła?" | zapobiega przedwczesnym restartom wolno startujących aplikacji |

Dla usługi AI readiness chroni ładowanie modelu i gotowość zależności; liveness to "zawór bezpieczeństwa" dla zakleszczeń i niekontrolowanego wzrostu pamięci. Zbyt agresywna liveness powoduje zbędne restarty i zimne starty. [unit 5] Typy sond w dokumentacji: `Liveness`, `Readiness`, `Startup`; protokół HTTP (`httpGet`) lub TCP (`tcpSocket`); `exec` nie jest wspierane. [docs: https://learn.microsoft.com/azure/container-apps/health-probes]

## D2. Typowe przyczyny porażek sond
_Źródło: [unit 5]_

Porażki sond to najczęściej błędy konfiguracji, nie kodu. Sprawdź najpierw: zły **port**, zła **ścieżka**, zbyt krótkie **limity czasu** (timeouts) dla ładowania modelu lub rozgrzewki oraz zależność od wywołania zewnętrznego przy starcie, które zawodzi lub jest wolne.

## D3. Konfiguracja sond w szablonie aplikacji
_Źródło: [unit 5]_

Sondy leżą obok definicji kontenera. Endpoint ma zwracać sukces dopiero, gdy usługa naprawdę jest gotowa.

```yaml
# Code fragment - focus on readiness and liveness probes
properties:
  template:
    containers:
    - name: api
      image: <registry>/<repo>@sha256:<digest>
      probes:
      - type: Readiness
        httpGet:
          path: /health/ready
          port: 8080
        initialDelaySeconds: 20
        periodSeconds: 10
        timeoutSeconds: 2
        failureThreshold: 3
      - type: Liveness
        httpGet:
          path: /health/live
          port: 8080
        initialDelaySeconds: 60
        periodSeconds: 10
        timeoutSeconds: 2
        failureThreshold: 3
```

## D4. Rozwiązywanie problemów z sondami podczas rolloutu
_Źródło: [unit 5]_

Łącz status rewizji i logi: status mówi, czy platforma uważa rewizję za zdrową, a logi, czy aplikacja wystartowała i jaki endpoint wystawiła. Endpointy sond nie powinny zależeć od zewnętrznych systemów, które mogą być niedostępne podczas incydentu. Jeśli usługa ładuje model przy starcie, dostrój `initialDelaySeconds` i `failureThreshold`. Gdy liveness zawodzi krótko po starcie, jest albo zbyt agresywna, albo proces szybko pada przez złą konfigurację.

## D5. Domyślne sondy i limity (z dokumentacji)
_Źródło: [docs: https://learn.microsoft.com/azure/container-apps/health-probes]_

Gdy włączysz ingress i nie zdefiniujesz danego typu sondy, platforma dodaje **domyślne sondy TCP** na porcie docelowym ingress (poza profilami GPU). Przy sondach TCP porty muszą się zgadzać z `targetPort` ingress. [docs: https://learn.microsoft.com/azure/container-apps/health-probes]

| Sonda | Domyślnie |
|---|---|
| Startup | TCP, timeout 3 s, period 1 s, initial delay 1 s, success 1, failure **240** |
| Readiness | TCP, timeout 5 s, period 5 s, initial delay 3 s, success 1, failure **48** |
| Liveness | TCP na porcie ingress (bez dodatkowych parametrów w tabeli) |

Limity wartości: `initialDelaySeconds` 1-60; `periodSeconds` 1-240 (domyślnie 10); `timeoutSeconds` 1-240 (domyślnie 1); `failureThreshold` 1-10 (domyślnie 3); `successThreshold` 1-10 (domyślnie 1, **musi być 1** dla liveness i startup). W trybie Single ruch przechodzi na nową rewizję automatycznie po sukcesie readiness; w trybie Multiple poczekaj na sukces readiness przed przesunięciem ruchu. Rewizja jest oznaczona jako niezdrowa, jeśli **którakolwiek** replika oblewa readiness. [docs: https://learn.microsoft.com/azure/container-apps/health-probes]

> Wersje sprawdzone 2026-11-22 (źródło: strony health-probes i szablonu `Microsoft.App/containerApps` w docs): wartości domyślne i limity jak wyżej; schemat sond zmienia się z wersjami API, więc moduł słusznie radzi sprawdzić aktualny schemat przed użyciem. Przed użyciem sprawdź https://learn.microsoft.com/azure/container-apps/health-probes.

(egzamin je lubi: readiness = ruch, liveness = restart; limit `initialDelaySeconds` to 60; `successThreshold` musi być 1 dla liveness i startup)

---

# CZĘŚĆ E - OPTYMALIZACJA ZASOBÓW I SKALOWANIA

## E1. Rozmiar repliki a całkowity koszt
_Źródło: [unit 6]_

Zasoby i skalowanie decydują, czy usługa AI jest responsywna i mieści się w budżecie. Za mało CPU = ograniczanie (**throttling**) i skoki opóźnień; za mało pamięci = restarty i powtarzane zimne starty. Container Apps skaluje przez dodawanie i usuwanie replik. Koszt całkowity to rozmiar repliki (CPU i pamięć) razy liczba replik potrzebna w szczycie. Dla API mierz opóźnienie pod współbieżnością i obserwuj throttling; dla workerów mierz przepustowość i obserwuj presję pamięci.

## E2. Zmiana CPU i pamięci
_Źródło: [unit 6]_

```azurecli
az containerapp update \
  --name <app-name> \
  --resource-group <resource-group> \
  --cpu <cores> \
  --memory <size>
```

Po zmianie sprawdź, czy opóźnienie spadło i czy rewizja jest stabilna pod obciążeniem (zmiana CPU/pamięci to zmiana w `template`, więc powstaje nowa rewizja). W tekście pomocy CLI przy tych parametrach podano zakresy **0.25-2.0** rdzeni oraz **0.5-4.0** z końcówką `Gi` (np. `1.0Gi`). [docs: https://learn.microsoft.com/cli/azure/containerapp?view=azure-cli-latest]

## E3. Skalowanie: synchroniczne API a worker w tle
_Źródło: [unit 6]_

API synchroniczne potrzebuje stabilnej gotowości i przewidywalnych opóźnień, więc często ustawia się **minimalną liczbę replik** (mniej zimnych startów). Worker w tle zwykle zniesie skalowanie do zera, jeśli jest sterowany zdarzeniami i potrafi bezpiecznie dokończyć pracę przy zmniejszaniu liczby replik. Rollout tworzy nowe repliki; przy wolnym starcie mogą dłużej nie być gotowe, więc **dopasuj skalowanie do konfiguracji sond**, żeby rollout nie wyglądał na nieudany tylko dlatego, że rozgrzewka trwa. Szczegóły reguł skalowania: moduł 06.

## E4. Dobre praktyki kosztu i wydajności
_Źródło: [unit 6]_

Optymalizacja jest iteracyjna: zmieniaj jedną zmienną naraz, żeby wiedzieć, co poprawiło przepustowość lub opóźnienie.

- **Size for the bottleneck:** zwiększ CPU przy throttlingu, pamięć przy restartach z braku pamięci (OOM).
- **Align concurrency with CPU:** przy dużej współbieżności sprawdź, czy CPU repliki ją wytrzyma.
- **Use minimum replicas intentionally:** minimum zmniejsza zimne starty, ale podnosi koszt bazowy. Dokumentacja: replika skalowana do zera nie generuje kosztu zasobów, a nieczynne repliki przy minimum >0 są liczone stawką idle. [docs: https://learn.microsoft.com/azure/container-apps/billing]
- **Reassess after model changes:** nowe wersje modeli zmieniają czas startu, zużycie pamięci i opóźnienia.

---

# CZĘŚĆ F - ĆWICZENIA

Przeczytaj części A-E. Każde ćwiczenie ma ten sam układ pól (przewodnik stylu §5). Ćwiczenia I1 i I2 tworzą płatne zasoby w chmurze: rób je w osobnych grupach zasobów i zawsze wykonuj **Sprzątanie**. Kod aplikacji i SDK jest w Pythonie; odpowiednik C# jest w ćwiczeniu z kodem (I3).

> Wersje sprawdzone 2026-11-22 (źródło: pypi.org, api.nuget.org, `az version`, testy lokalne): Python 3.12; azure-mgmt-appcontainers 5.1.0; azure-identity 1.27.0; azure-monitor-query **2.0.0** (2025-07-30); C#: .NET SDK 10.0, Azure.ResourceManager.AppContainers 1.6.0, Azure.Identity 1.21.0, Azure.Monitor.Query.Logs **1.0.0** (nowy pakiet; starszy Azure.Monitor.Query ma 1.7.1); Azure CLI 2.90.0, rozszerzenie containerapp 1.3.0b6 (beta). Przed użyciem sprawdź PyPI i NuGet.

## I1. Diagnoza brakującej zmiennej, złego portu i historii logów (lab modułu)
(rozwinięcie jednostki 7; scenariusz: [unit 7])

**Gdzie / Plik:** terminal z Azure CLI (Bash), katalog z plikami `api/main.py`, `api/requirements.txt`, `api/Dockerfile` z ćwiczenia I1 w `04-deploy-containers-azure-container-apps.md` (pełny kod jest tam).

> Uwaga o labie: lab Microsoftu startuje od skryptu `azdeploy.py` z paczki `aca-manage-python.zip` (opcje menu 1: rejestr i obraz, 2: środowisko, 3: aplikacja z sekretami); tutaj te kroki robisz ręcznie. Lab wymaga planu Pay-As-You-Go lub innego płatnego (zadania ACR Tasks wstrzymane dla darmowych kredytów) [unit 7].

Przygotowanie (zmienne ustaw raz, w jednej sesji):
```bash
RG=rg-aca-manage-i1; LOC=westeurope
ACR_NAME=acrmanage$RANDOM$RANDOM; ACA_ENVIRONMENT=aca-env-i1; CONTAINER_APP_NAME=ai-api; MODEL_NAME=gpt-5.4-mini
az extension add --name containerapp --upgrade; az extension add --name log-analytics
az provider register --namespace Microsoft.App
az provider register --namespace Microsoft.OperationalInsights
az provider register --namespace Microsoft.ContainerRegistry

az group create --name $RG --location $LOC
az acr create --resource-group $RG --name $ACR_NAME --sku Basic --admin-enabled false
az acr build --registry $ACR_NAME --image ai-api:v1 --file api/Dockerfile api
az containerapp env create --name $ACA_ENVIRONMENT --resource-group $RG --location $LOC
az containerapp create --name $CONTAINER_APP_NAME --resource-group $RG --environment $ACA_ENVIRONMENT \
    --image $ACR_NAME.azurecr.io/ai-api:v1 --ingress external --target-port 8000 \
    --env-vars MODEL_NAME=$MODEL_NAME \
    --registry-server $ACR_NAME.azurecr.io --registry-identity system
FQDN=$(az containerapp show -n $CONTAINER_APP_NAME -g $RG --query properties.configuration.ingress.fqdn -o tsv)
curl -s https://$FQDN/            # model powinien byc "gpt-5.4-mini"
```
Awarie (po kolei):
```bash
# --- Awaria 1: brak zmiennej MODEL_NAME ---
az containerapp update -n $CONTAINER_APP_NAME -g $RG --remove-env-vars MODEL_NAME          # (1)
az containerapp revision list -n $CONTAINER_APP_NAME -g $RG -o table                       # (2)
curl -s https://$FQDN/                                                                     # (3)
az containerapp show -n $CONTAINER_APP_NAME -g $RG \
    --query "properties.template.containers[0].env" -o table                               # (4)
az containerapp update -n $CONTAINER_APP_NAME -g $RG --set-env-vars MODEL_NAME=$MODEL_NAME # (5)

# --- Awaria 2: zly port docelowy ingress ---
az containerapp ingress update -n $CONTAINER_APP_NAME -g $RG --target-port 3000            # (6)
curl -s https://$FQDN/health                                                               # (7)
az containerapp show -n $CONTAINER_APP_NAME -g $RG \
    --query "properties.configuration.ingress" -o yaml                                     # (8)
az containerapp logs show -n $CONTAINER_APP_NAME -g $RG                                    # (9)
az containerapp ingress update -n $CONTAINER_APP_NAME -g $RG --target-port 8000            # (10)
curl -s https://$FQDN/health

# --- Historia logow (KQL) ---
WORKSPACE_ID=$(az containerapp env show -n $ACA_ENVIRONMENT -g $RG \
    --query properties.appLogsConfiguration.logAnalyticsConfiguration.customerId -o tsv)   # (11)
az monitor log-analytics query -w $WORKSPACE_ID \
    --analytics-query "ContainerAppConsoleLogs_CL | where ContainerAppName_s == '$CONTAINER_APP_NAME' | project TimeGenerated, Log_s | order by TimeGenerated desc | take 20" -o table
az monitor log-analytics query -w $WORKSPACE_ID \
    --analytics-query "ContainerAppConsoleLogs_CL | where ContainerAppName_s == '$CONTAINER_APP_NAME' and Log_s contains 'error' | order by TimeGenerated desc | take 20" -o table
```
Objaśnienie: (1) usunięcie zmiennej tworzy nową rewizję (zmiana w `template`); (2) nowa rewizja z wyższym sufiksem, np. `ai-api--0000002`, ma `TrafficWeight` 100 [unit 7]; (3) `model` zmienia się na wartość domyślną `not-configured` (to zachowanie aplikacji z `os.getenv(..., "not-configured")`); (4) potwierdzenie braku zmiennej w konfiguracji; (5) naprawa; (6) zmiana portu docelowego; (7) żądanie zawodzi; (8) w ingress widać `targetPort: 3000`; (9) logi pokazują `gunicorn` nasłuchujący na porcie 8000, co dowodzi niezgodności; (10) naprawa; (11) `customerId` obszaru Log Analytics. [unit 7] Opcjonalnie `| jq .model` (narzędzie `jq` filtruje JSON) przy wywołaniu `/`.

Typowe problemy z labu [unit 7]: aplikacja nie odpowiada - sprawdź `az containerapp revision list` (czy rewizja jest aktywna) i `az containerapp show` (konfiguracja ingress); brak logów - logi konsoli z CLI są tylko niedawne, a dane w Log Analytics pojawiają się po 2-5 minutach; zmienne nie działają - zmiana zmiennych tworzy nową rewizję, więc sprawdź, czy jest aktywna, i pamiętaj, że `--replace-env-vars` zastępuje **wszystkie** zmienne, nie tylko wskazane.

**Koszt:** jak w `04-deploy-containers-azure-container-apps.md` I1: ACR Basic naliczany dziennie (stawka: Niesprawdzone - zweryfikuj w https://azure.microsoft.com/pricing/details/container-registry/); Container Apps plan Consumption ma miesięczne darmowe progi 180 000 vCPU-s, 360 000 GiB-s i 2 mln żądań; Log Analytics rozlicza ilość logów [docs: https://learn.microsoft.com/azure/container-apps/billing]. Kilka godzin pracy to grosze-kilka złotych po Sprzątaniu.

**Zadania:** (a) wykonaj przygotowanie i obie awarie; (b) **zepsuj to:** w awarii 2 zmierz, czy zmiana `--target-port` utworzyła nową rewizję (porównaj `revision list` przed i po); (c) zapytaj KQL o logi konsoli; poczekaj kilka minut, jeśli tabela jest jeszcze pusta; (d) porównaj wynik KQL z wynikiem `az containerapp logs show`.

**Sprawdź:** (a) po awarii 1 `/` pokazuje `"model":"not-configured"`, a po naprawie `"gpt-5.4-mini"`; po awarii 2 `/health` nie zwraca `{"status":"healthy"}`, a po naprawie tak; (b) `revision list` nie pokazuje nowej rewizji po zmianie portu, bo ingress to zmiana application-scope [docs: https://learn.microsoft.com/azure/container-apps/revisions]; (c) tabela zwraca wiersze z `Log_s` (komunikaty `gunicorn` i żądań) [unit 7]; (d) `logs show` daje krótki, niedawny fragment, a KQL historię (limit `take 20` w obu).

**Sprzątanie:**
```bash
az group delete --name $RG --yes --no-wait
az group exists --name $RG      # po kilku minutach powinno wypisać false
```

## I2. Sondy zdrowia w YAML i operacje cyklu życia
(rozwinięcie sekcji A3-A4, B1-B3, D3-D5)

**Gdzie / Plik:** terminal z Azure CLI (Bash); plik `probes.yml`; publiczny obraz quickstart (nie potrzeba ACR).

```bash
RG=rg-aca-manage-i2; LOC=westeurope
az group create --name $RG --location $LOC
az containerapp env create --name aca-env-i2 --resource-group $RG --location $LOC
ENV_ID=$(az containerapp env show --name aca-env-i2 --resource-group $RG --query id -o tsv)
echo $ENV_ID     # wklej do probes.yml w miejsce <ID srodowiska>
```
`probes.yml`:
```yaml
location: westeurope
name: probe-demo
type: Microsoft.App/containerApps
properties:
  managedEnvironmentId: <ID środowiska>
  configuration:
    activeRevisionsMode: Multiple
    ingress:
      external: true
      targetPort: 80
  template:
    containers:
    - name: web
      image: mcr.microsoft.com/k8se/quickstart:latest
      probes:
      - type: Readiness
        httpGet:
          path: /
          port: 80
        initialDelaySeconds: 5
        periodSeconds: 5
        failureThreshold: 3
      - type: Liveness
        httpGet:
          path: /
          port: 80
        initialDelaySeconds: 10
        periodSeconds: 10
        failureThreshold: 3
    scale:
      minReplicas: 1
      maxReplicas: 1
```
```bash
az containerapp create --name probe-demo --resource-group $RG --yaml ./probes.yml                      # (1)
az containerapp revision list -n probe-demo -g $RG --query "[].{name:name,active:properties.active,health:properties.healthState}" -o table   # (2)

# zepsuj: zmien sciezke readiness w probes.yml z "/" na "/nie-ma-takiej" i zastosuj
az containerapp update --name probe-demo --resource-group $RG --yaml ./probes.yml                      # (3)

az containerapp revision deactivate -n probe-demo -g $RG --revision <zla-rewizja>                      # (4)
az containerapp revision activate   -n probe-demo -g $RG --revision <zla-rewizja>                      # (5)
az containerapp revision restart    -n probe-demo -g $RG --revision <dobra-rewizja>                    # (6)
az containerapp stop  -n probe-demo -g $RG                                                             # (7)
az containerapp start -n probe-demo -g $RG
```
Objaśnienie: (1) tryb `Multiple`, żeby obie rewizje mogły współistnieć; sondy HTTP na porcie 80; (2) kolumny `active` i `health` z zapytania z jednostki 3; (3) nowa rewizja z błędną ścieżką readiness; (4)-(5) dezaktywacja i ponowna aktywacja; (6) restart jednej rewizji; (7) stop i start całej aplikacji.

**Koszt:** replika z `minReplicas: 1` działa ciągle w planie Consumption: 0,25 vCPU i 0,5 GiB to około 21 600 vCPU-s i 43 200 GiB-s na dobę (domyślny rozmiar repliki sprawdź w `az containerapp show`: Niesprawdzone), czyli kilka dni mieści się w darmowych progach miesięcznych (180 000 vCPU-s, 360 000 GiB-s) [docs: https://learn.microsoft.com/azure/container-apps/billing]; Log Analytics rozlicza ilość logów (pomijalna). Mimo to wykonaj Sprzątanie.

**Zadania:** (a) wdróż YAML i sprawdź, że rewizja jest zdrowa; (b) **zepsuj to:** zmień ścieżkę readiness na nieistniejącą i zastosuj YAML; obserwuj `healthState` nowej rewizji i ruch; (c) zdeaktywuj uszkodzoną rewizję i sprawdź kolumnę `active`; (d) aktywuj ją ponownie, a potem zrestartuj dobrą rewizję; (e) zatrzymaj i uruchom aplikację.

**Sprawdź:** (a) rewizja ma `active = True` i stan zdrowia dobry (nazwa wartości w kolumnie `health` zależy od API: obserwuj różnicę między rewizjami) [autor: niezweryfikowane]; (b) nowa rewizja nie staje się zdrowa, bo readiness oblewa sprawdzenie (rewizja jest oznaczona jako niezdrowa, gdy replika oblewa readiness) [docs: https://learn.microsoft.com/azure/container-apps/health-probes]; (c) `active` zmienia się na `False`; (d) `active` wraca na `True`, a restart nie tworzy nowej rewizji (to restart tej samej) [autor: niezweryfikowane]; (e) po `stop` brak działających replik, po `start` aplikacja wraca [unit 3].

**Sprzątanie:**
```bash
az group delete --name $RG --yes --no-wait
az group exists --name $RG
```

## I3. Rewizje i logi KQL przez SDK: Python i odpowiednik C# (.NET)
(rozwinięcie sekcji A4, C4; wymaga aplikacji z I1 przed jej Sprzątaniem)

**Gdzie / Plik:** `manage.py` (Python) i `Program.cs` (C#); `az login`; obszar Log Analytics z I1.

```bash
pip install azure-mgmt-appcontainers==5.1.0 azure-monitor-query==2.0.0 azure-identity==1.27.0
export AZURE_SUBSCRIPTION_ID=$(az account show --query id -o tsv)
export RESOURCE_GROUP=rg-aca-manage-i1 CONTAINER_APP_NAME=ai-api
export WORKSPACE_ID=$(az containerapp env show -n aca-env-i1 -g $RESOURCE_GROUP \
    --query properties.appLogsConfiguration.logAnalyticsConfiguration.customerId -o tsv)
```
`manage.py`:
```python
import os
import sys
from datetime import timedelta

from azure.identity import DefaultAzureCredential
from azure.mgmt.appcontainers import ContainerAppsAPIClient
from azure.monitor.query import LogsQueryClient, LogsQueryStatus

subscription_id = os.environ["AZURE_SUBSCRIPTION_ID"]
resource_group = os.environ["RESOURCE_GROUP"]
app_name = os.environ["CONTAINER_APP_NAME"]
workspace_id = os.environ["WORKSPACE_ID"]  # customerId obszaru Log Analytics

credential = DefaultAzureCredential()
aca = ContainerAppsAPIClient(credential, subscription_id)  # (1)

# (2) Diagnoza: ktora rewizja jest zdrowa i ma ruch
for rev in aca.container_apps_revisions.list_revisions(resource_group, app_name):
    p = rev.properties
    print(rev.name, "active =", p.active, "traffic =", p.traffic_weight, "health =", p.health_state)

# (3) Opcjonalnie: restart wskazanej rewizji (np. po zmianie wartosci sekretu)
if len(sys.argv) > 1:
    aca.container_apps_revisions.restart_revision(resource_group, app_name, sys.argv[1])
    print("restart zlecony dla", sys.argv[1])

# (4) Historia logow z Log Analytics (KQL)
logs = LogsQueryClient(credential)
kql = (
    f"ContainerAppConsoleLogs_CL | where ContainerAppName_s == '{app_name}' "
    "| project TimeGenerated, RevisionName_s, Log_s | order by TimeGenerated desc | take 20"
)
result = logs.query_workspace(workspace_id, kql, timespan=timedelta(hours=1))  # (5)
if result.status == LogsQueryStatus.SUCCESS:
    for row in result.tables[0].rows:  # (6)
        print(row[0], row[1], row[2])
else:
    print("wynik czesciowy:", result.partial_error)
```
Objaśnienie: (1) klient zarządzania, ten sam wzorzec co w module 04 (`04-deploy-containers-azure-container-apps.md` I3); (2) lista rewizji ze stanem; (3) `restart_revision` odpowiada poleceniu `az containerapp revision restart`; (4) KQL jak w C4; (5) `query_workspace` wymaga argumentu `timespan` (przedział czasu); (6) kolejność kolumn w wierszu wynika z `project`. Skrypt skompilowano (`py_compile`), a nazwy metod (`restart_revision`, `deactivate_revision`, `activate_revision`, sygnaturę `query_workspace`) sprawdzono w zainstalowanych pakietach; nie uruchamiano go na prawdziwej subskrypcji [autor: test 2026-11-22].

**Odpowiednik C# (.NET)**

Różnice: logi zapytasz klientem `LogsQueryClient` z pakietu **Azure.Monitor.Query.Logs** (1.0.0; dawniej pakiet Azure.Monitor.Query), a przedział czasu to `LogsQueryTimeRange`; wiersze czytasz po nazwie kolumny (`row["Log_s"]`), nie po indeksie. Restart rewizji to metoda `RestartRevisionAsync()` na zasobie rewizji. Projekt: `dotnet new console -f net10.0`, `dotnet add package Azure.ResourceManager.AppContainers --version 1.6.0`, `dotnet add package Azure.Monitor.Query.Logs --version 1.0.0`, `dotnet add package Azure.Identity --version 1.21.0`.
```csharp
using Azure.Identity;
using Azure.Monitor.Query.Logs;
using Azure.ResourceManager;
using Azure.ResourceManager.AppContainers;

string sub = Environment.GetEnvironmentVariable("AZURE_SUBSCRIPTION_ID")!;
string rg = Environment.GetEnvironmentVariable("RESOURCE_GROUP")!;
string appName = Environment.GetEnvironmentVariable("CONTAINER_APP_NAME")!;
string workspaceId = Environment.GetEnvironmentVariable("WORKSPACE_ID")!;

var credential = new DefaultAzureCredential();
var arm = new ArmClient(credential);
ContainerAppResource app = arm.GetContainerAppResource(
    ContainerAppResource.CreateResourceIdentifier(sub, rg, appName));

// Diagnoza: ktora rewizja jest zdrowa i ma ruch
await foreach (ContainerAppRevisionResource rev in app.GetContainerAppRevisions().GetAllAsync())
    Console.WriteLine($"{rev.Data.Name} active={rev.Data.IsActive} traffic={rev.Data.TrafficWeight} health={rev.Data.HealthState}");

// Opcjonalnie: restart wskazanej rewizji (nazwa jako pierwszy argument)
if (args.Length > 0)
{
    ContainerAppRevisionResource target = await app.GetContainerAppRevisionAsync(args[0]);
    await target.RestartRevisionAsync();
    Console.WriteLine($"restart zlecony dla {args[0]}");
}

// Historia logow z Log Analytics (KQL)
var logs = new LogsQueryClient(credential);
string kql = $"ContainerAppConsoleLogs_CL | where ContainerAppName_s == '{appName}' " +
             "| project TimeGenerated, RevisionName_s, Log_s | order by TimeGenerated desc | take 20";
var response = await logs.QueryWorkspaceAsync(workspaceId, kql, new LogsQueryTimeRange(TimeSpan.FromHours(1)));
foreach (var row in response.Value.Table.Rows)
    Console.WriteLine($"{row["TimeGenerated"]}  {row["RevisionName_s"]}  {row["Log_s"]}");
```
[docs: https://learn.microsoft.com/dotnet/api/overview/azure/resourcemanager.appcontainers-readme?view=azure-dotnet] [autor: test 2026-11-22] (kod skompilowano na .NET SDK 10.0.401 z pakietami jak wyżej; nie uruchamiano go na prawdziwej subskrypcji).

**Koszt:** brak kosztu własnego (odczyt i restart); koszt aplikacji i Log Analytics jak w I1.

**Zadania:** (a) uruchom `python manage.py` i porównaj wynik z `az containerapp revision list`; (b) **zepsuj to:** uruchom `python manage.py nie-ma-takiej-rewizji` i przeczytaj błąd; (c) uruchom wersję C# (`dotnet run`) i porównaj kolumny z KQL z I1.

**Sprawdź:** (a) te same rewizje i stan aktywności co w CLI; (b) błąd "nie znaleziono" dla nazwy rewizji (dokładny komunikat zależy od wersji SDK) [autor: niezweryfikowane]; (c) te same komunikaty `Log_s`; tabela może być pusta kilka minut po wdrożeniu [docs: https://learn.microsoft.com/azure/container-apps/log-monitoring].

**Sprzątanie:** brak (skrypty tylko czytają i restartują); po I3 wykonaj Sprzątanie I1.

---

# CZĘŚĆ G - QUIZ Z MODUŁU MICROSOFTU (5 pytań) + wyjaśnienia

Learn nie pokazuje poprawnych odpowiedzi w ocenie modułu, więc odpowiedzi są wnioskowane z treści jednostek; przy każdym pytaniu jest krótki znacznik.

## G1. Najlepsza identyfikowalność nowego obrazu na produkcji
-> *Odwołanie po digeście (`myregistry.azurecr.io/app@sha256:<digest>`).* Tag `latest` nie jest niezmienny, a przebudowa lokalnie w każdym środowisku nie gwarantuje tego samego artefaktu. *(odpowiedź wnioskowana: unit 2, sekcja "Use tags and digests intentionally")*

## G2. Zdjęcie rewizji z ruchu bez kasowania
-> *`az containerapp revision deactivate`.* `revision delete` kasuje rewizję, a `az containerapp stop` zatrzymuje całą aplikację. *(odpowiedź wnioskowana: unit 2, sekcja "List, investigate, and manage revisions")*

## G3. Rewizja oblewa readiness zaraz po wdrożeniu - najczęstsza przyczyna
-> *Sonda readiness wskazuje zły port lub ścieżkę serwera HTTP kontenera.* Zbyt "mały" obraz i brak routingu do internetu nie są typowymi przyczynami porażki readiness. *(odpowiedź wnioskowana: unit 5, sekcja "Recognize common probe failure causes")*

## G4. Podejrzewasz wyjątek tylko w jednej rewizji - pierwszy krok
-> *Strumieniować logi aplikacji i filtrować po rewizji, odtwarzając żądanie.* Kasowanie starszych rewizji i zwiększanie CPU/pamięci przed badaniem to nie jest potwierdzenie problemu. *(odpowiedź wnioskowana: unit 4, sekcje "Use log streaming" i "Troubleshoot revision-specific failures")*

## G5. Wysokie opóźnienie i throttling CPU w szczycie
-> *Zwiększyć CPU na replikę, a potem ponownie ocenić reguły skalowania względem przepustowości.* Dezaktywacja najnowszej rewizji nie rozwiązuje throttlingu, a zamiana digestu na tag nie ma związku z CPU. *(odpowiedź wnioskowana: unit 6, sekcja "Best practices for cost and performance: Size for the bottleneck")*

---

# CZĘŚĆ H - PYTANIA KONTROLNE (odpowiedz bez zaglądania)

## H1. Czym różni się tag od digestu i którego użyć na produkcji?
<details><summary>Odpowiedź</summary>

Tag jest zmienną etykietą (może wskazywać różne obrazy), digest (`sha256:...`) jest niezmiennym identyfikatorem zawartości. Na produkcji używaj digestu dla identyfikowalności.
</details>

## H2. Czym różni się `revision deactivate` od `revision delete` i `containerapp stop`?
<details><summary>Odpowiedź</summary>

`deactivate` zdejmuje rewizję z ruchu bez kasowania (zachowuje dowody i rollback); `delete` usuwa rewizję; `stop` zatrzymuje całą aplikację, więc jest zbyt szerokie, gdy zła jest jedna rewizja.
</details>

## H3. Ile nieaktywnych rewizji zachowuje się domyślnie i jak zmienić próg?
<details><summary>Odpowiedź</summary>

Domyślnie 100; zmiana przez `--max-inactive-revisions` (w teście lokalnym oznaczone jako Preview).
</details>

## H4. Co robi platforma, gdy oblewa sonda readiness, a co przy liveness?
<details><summary>Odpowiedź</summary>

Readiness: replika nie dostaje ruchu. Liveness: replika jest restartowana. Rewizja jest oznaczona jako niezdrowa, gdy jakakolwiek replika oblewa readiness.
</details>

## H5. Wymień pięć typowych wzorców awarii rewizji z modułu.
<details><summary>Odpowiedź</summary>

Błędy pobierania obrazu, niezgodność portów, brak konfiguracji (zmienne/sekrety), błędy sond, presja zasobów (limit pamięci, throttling CPU).
</details>

## H6. Które limity sond warto znać?
<details><summary>Odpowiedź</summary>

`initialDelaySeconds` 1-60; `periodSeconds` 1-240 (domyślnie 10); `timeoutSeconds` 1-240 (domyślnie 1); `failureThreshold` 1-10 (domyślnie 3); `successThreshold` musi wynosić 1 dla liveness i startup.
</details>

## H7. Jak sprawdzić historię logów konsoli aplikacji sprzed tygodnia?
<details><summary>Odpowiedź</summary>

W obszarze Log Analytics środowiska: tabela `ContainerAppConsoleLogs_CL` (kolumny `ContainerAppName_s`, `RevisionName_s`, `Log_s`), np. przez `az monitor log-analytics query`. `az containerapp logs show` pokazuje tylko niedawne logi.
</details>

## H8. Co zwiększysz przy throttlingu CPU, a co przy restartach z braku pamięci?
<details><summary>Odpowiedź</summary>

Throttling CPU: więcej CPU na replikę; restarty z braku pamięci: więcej pamięci. Zmieniaj jedną zmienną naraz i oceniaj ponownie reguły skalowania.
</details>

---

# CZĘŚĆ J - CO DALEJ

## J1. Następny krok i pokrycie egzaminu
_Źródło: [moduł kursu: scale-containers-azure-container-apps] + [moduł kursu: dynamic-sessions-azure-container-apps]_

Dalej: moduł 06 (reguły skalowania, KEDA, rewizje i podział ruchu), moduł 07 (dynamic sessions). Wróć do części D, gdy rozwiązujesz zadania o sondach, i do C4, gdy piszesz zapytania KQL o logi.

**Pokrycie egzaminu** (wiersze z `sources/_exam/skill-map.md`, stan study guide z 2026-10-30):

| Umiejętność (study guide) | Domena | Status | Gdzie |
|---|---|---|---|
| Deploy applications to Azure Container Apps, including environment configuration and revision management | 1 | uczone tutaj (zarządzanie rewizjami, część A); wdrożenie i środowisko w module 04 | `05-manage-containers-azure-container-apps.md` §A1-§A5; `04-deploy-containers-azure-container-apps.md` |
| Monitor and troubleshoot solutions on AKS and Container Apps by inspecting logs, events, and end-to-end connectivity | 1 | uczone tutaj (część Container Apps: §B4, §C1-§C4, §D4); część AKS w module monitor-apps-azure-kubernetes-service | `05-manage-containers-azure-container-apps.md` §C1-§C4; https://learn.microsoft.com/en-us/training/modules/monitor-apps-azure-kubernetes-service/ |
| Implement event-driven scaling by using KEDA in Container Apps | 1 | uczone w module 06 (tu tylko wzmianka o skalowaniu, §E3) | https://learn.microsoft.com/en-us/training/modules/scale-containers-azure-container-apps/ |
| Write KQL queries to analyze logs and metrics | 4 | uczone w module analyze-telemetry-logs-metrics (tu tylko proste zapytania, §C4) | https://learn.microsoft.com/en-us/training/modules/analyze-telemetry-logs-metrics/ |

## J2. Uczciwie o egzaminie
_Źródło: [autor]_

Ten moduł pokrywa zarządzanie rewizjami, lifecycle, logi, sondy i zasoby w Container Apps. Nie uczy KEDA ani podziału ruchu (moduł 06) ani pełnego KQL (osobny moduł kursu). Ćwicz diagnozę: rozpoznawanie, czy awaria to obraz, port, konfiguracja, sonda czy zasoby, oraz różnicę revision-scope vs application-scope z modułu 04. Quiz jest krótki (5 pytań), więc egzamin będzie głębszy: szczególnie limity i domyślne wartości sond oraz polecenia `deactivate`, `restart`, `stop`.
