# Moduł 06: Skalowanie kontenerów w Azure Container Apps (reguły skalowania, KEDA, zasoby, tryby rewizji, podział ruchu)

Moduł Microsoft Learn: **Scale containers in Azure Container Apps** - https://learn.microsoft.com/en-us/training/modules/scale-containers-azure-container-apps/
Cel: po tym module umiesz skonfigurować reguły skalowania HTTP, TCP, CPU i pamięci oraz oparte na KEDA reguły zdarzeniowe (Service Bus, Storage Queue, Event Hubs, Kafka, Redis, cron, Prometheus), uwierzytelnić je tożsamością zarządzaną, dobrać CPU i pamięć repliki oraz użyć trybów rewizji, podziału ruchu i etykiet.
Czas: ok. 4-5 h, orientacyjnie (moduł Learn ma ok. 87 min wg katalogu, w tym ćwiczenie ok. 30 min; tutaj dochodzą eksperymenty i ćwiczenie z SDK). Wymagania: Azure CLI, Python 3.12; ćwiczenia I1-I2 tworzą płatne zasoby w chmurze (ACR, środowisko Container Apps, Log Analytics); koszty w polach **Koszt** przy ćwiczeniach.

Źródła: `[unit N]` = jednostka modułu Learn, `[docs: URL]` = dokumentacja produktu, `[autor]` = dodane przez autora; pełna legenda: przewodnik stylu §3. "Moduł N" znaczy plik tutoriala `0N-*.md`.

---

# CZĘŚĆ A - REGUŁY SKALOWANIA

## A1. Definicja skalowania: limity, reguły, zachowanie
_Źródło: [unit 2] + [docs: https://learn.microsoft.com/azure/container-apps/scale-app]_

**Skalowanie poziome** (horizontal scaling) to dodawanie i usuwanie **replik** (kopii kontenera), a nie powiększanie jednej kopii. Definicja skalowania ma trzy części:

| Część | Co określa |
|---|---|
| **Limity** (limits) | minimalna i maksymalna liczba replik |
| **Reguły** (rules) | wyzwalacze (triggers), czyli kiedy skalować |
| **Zachowanie** (behavior) | jak platforma podejmuje decyzje w czasie: odstępy, algorytm |

Pod spodem pracuje **KEDA** (Kubernetes Event-driven Autoscaling, czyli skalowanie sterowane zdarzeniami): platforma tłumaczy Twoje ustawienia na specyfikacje KEDA, które obserwują wyzwalacze i zmieniają liczbę replik. [unit 2]

**Wartości domyślne** (egzamin je lubi): gdy ingress jest włączony i nie ma własnych reguł, aplikacja skaluje się od **0 do 10** replik (domyślna reguła HTTP). Gdy ingress jest **wyłączony**, a nie ustawisz ani minimum, ani własnej reguły, aplikacja skaluje się do zera i **nie ma jak się ponownie uruchomić** (brak wyzwalacza). Aby tego uniknąć, ustaw minimum na 1 lub więcej albo dodaj regułę. [unit 2] [docs: https://learn.microsoft.com/azure/container-apps/scale-app]

Rozliczanie zależy od liczby replik: przy skalowaniu do zera nie płacisz za obliczenia, repliki działające bez przetwarzania żądań są liczone niższą stawką **idle**, a minimum >= 1 zapewnia dostępność kosztem wyższego kosztu bazowego. [unit 2]

## A2. Reguła HTTP
_Źródło: [unit 2]_

Skalowanie HTTP zmienia liczbę replik według liczby **równoległych żądań HTTP** (concurrent requests). Platforma liczy żądania z ostatnich 15 sekund i dzieli przez 15; gdy wynik przekracza próg, tworzy kolejne repliki. Domyślny próg to **10 żądań na replikę**. Niższy próg skaluje wcześniej (więcej zapasu, ale może powstać zbędnych replik), wyższy lepiej wykorzystuje replikę, ale zwiększa opóźnienia, zanim pojawią się nowe repliki.

Reguła HTTP pasuje do synchronicznych API i aplikacji webowych, **obsługuje skalowanie do zera**. Zadania (jobs) Container Apps nie mogą jej używać, bo nie wystawiają HTTP.

```azurecli
az containerapp create \
  --name order-api \
  --resource-group rg-ecommerce \
  --environment my-environment \
  --image myregistry.azurecr.io/order-api:v1 \
  --min-replicas 1 \
  --max-replicas 10 \
  --scale-rule-name http-scaling \
  --scale-rule-type http \
  --scale-rule-http-concurrency 50
```

## A3. Reguła TCP
_Źródło: [unit 2]_

Skalowanie TCP opiera się na liczbie **równoległych połączeń TCP**; jest przeznaczone dla aplikacji z trwałymi połączeniami (zamiast krótkich cykli żądanie-odpowiedź). Lepsze niż HTTP dla serwerów WebSocket, usług gRPC i aplikacji z pulami połączeń do baz danych. Okno uśredniania to te same 15 sekund. Parametr: `--scale-rule-tcp-concurrency`. Reguła TCP także obsługuje skalowanie do zera: gdy wszystkie połączenia się zamkną, po okresie cool-down aplikacja może zejść do zera, a nowe połączenia uruchamiają skalowanie w górę.

## A4. Reguły CPU i pamięci
_Źródło: [unit 2]_

Reguły CPU i pamięci zmieniają liczbę replik według **średniego wykorzystania** zasobów (próg w procentach). Są zaimplementowane jako niestandardowe scalery KEDA. **Ograniczenie krytyczne (egzamin je lubi):** reguły CPU i pamięci **nie potrafią skalować do zera**, bo potrzeba co najmniej jednej działającej repliki, żeby zmierzyć wykorzystanie. Gdy potrzebujesz skalowania do zera, połącz je z regułą HTTP lub zdarzeniową albo użyj HTTP jako głównego wyzwalacza.

CPU pasuje do zadań obliczeniowych (obrazy, wideo, inferencja ML), pamięć do cache'owania i agregacji dużych zbiorów. Fragment YAML łączący HTTP i CPU (skalowanie, gdy współbieżność HTTP > 100 **lub** CPU > 70%):

```yaml
# Code fragment - focus on scale rules configuration
scale:
  minReplicas: 1
  maxReplicas: 20
  rules:
    - name: http-scaling
      http:
        metadata:
          concurrentRequests: "100"
    - name: cpu-scaling
      custom:
        type: cpu
        metadata:
          type: Utilization
          value: "70"
```

## A5. Zachowanie skalowania: odstępy, okna i kroki
_Źródło: [unit 2] + [docs: https://learn.microsoft.com/azure/container-apps/scale-app]_

| Parametr | Wartość | Znaczenie |
|---|---|---|
| Polling interval | **30 s** | jak często KEDA odpytuje źródła zdarzeń; dla scalerów niestandardowych (CPU, pamięć, zdarzenia); reguły HTTP i TCP używają okna 15 s |
| Cool-down period | **300 s** (5 min) | jak długo czekać po ostatnim zdarzeniu, zanim aplikacja zejdzie do zera |
| Scale-up stabilization window | **0 s** | skalowanie w górę może ruszyć od razu |
| Scale-down stabilization window | **300 s** | tyle czeka po wykryciu warunków zejścia, zanim usunie repliki |
| Scale-up step | **1, 4, 8, 16, 32, ...** | do maksimum; podwajanie kroku |
| Scale-down step | **100%** replik do wyłączenia | wszystkie nadmiarowe repliki naraz |
| Algorytm | `desiredReplicas = ceil(currentMetricValue / targetMetricValue)` | wzór z dokumentacji |

Dokumentacja precyzuje: cool-down działa tylko przy schodzeniu **z ostatniej repliki do zera**; przy usuwaniu pozostałych replik nie ma wpływu, a polling nie dotyczy reguł HTTP i TCP. [docs: https://learn.microsoft.com/azure/container-apps/scale-app] W YAML-u pola to `cooldownPeriod` i `pollingInterval` w sekcji `scale`. [unit 7]

## A6. Dobre praktyki
_Źródło: [unit 2]_

- **Ustaw sensowne minimum replik:** na produkcji co najmniej 1 (brak zimnych startów); do zera tylko dev i obciążenia naprawdę sporadyczne.
- **HTTP dla API:** najbardziej responsywne dla synchronicznych wzorców i obsługuje skalowanie do zera.
- **Łącz kilka reguł:** platforma skaluje w górę, gdy **którakolwiek** reguła przekroczy próg.
- **Uwzględnij cool-down:** 300 s przed zejściem do zera podnosi koszt przy krótkich impulsach.

---

# CZĘŚĆ B - SKALOWANIE ZDARZENIOWE (KEDA)

## B1. Integracja z KEDA i scalery Microsoftu
_Źródło: [unit 3]_

Skalowanie zdarzeniowe reaguje na sygnały spoza HTTP: kolejki wiadomości, strumienie zdarzeń. KEDA obserwuje źródło (głębokość kolejki, opóźnienie partycji, własne metryki), a Container Apps tłumaczy konfigurację scalera na regułę. **Scalery niestandardowe** (custom) wzorują się na specyfikacji KEDA `ScaledObject`: podajesz typ scalera, metadane (połączenie i progi) oraz uwierzytelnianie; platforma odpytuje źródło co 30 s.

Microsoft utrzymuje scalery z pierwszoplanowym wsparciem: **Azure Service Bus, Azure Event Hubs, Azure Storage Queue, Azure Blob Storage, Azure Log Analytics i Azure Monitor**. Scalery społecznościowe obsługują źródła spoza Azure. Skalowanie zdarzeniowe **obsługuje skalowanie do zera**: bez wiadomości 0 replik, nowe wiadomości uruchamiają skalowanie.

## B2. Azure Service Bus
_Źródło: [unit 3]_

Scaler Service Bus zmienia repliki według liczby wiadomości w kolejce lub w subskrypcji tematu: **liczba aktywnych wiadomości / `messageCount`** daje żądaną liczbę replik (przy `messageCount` 5 i 50 wiadomościach: 10 replik). Metadane: `queueName` (albo `topicName` + `subscriptionName`), `namespace`, `messageCount`.

```azurecli
az containerapp create \
  --name order-processor \
  --resource-group rg-ecommerce \
  --environment my-environment \
  --image myregistry.azurecr.io/order-processor:v1 \
  --min-replicas 0 \
  --max-replicas 30 \
  --secrets "sb-connection=<SERVICE_BUS_CONNECTION_STRING>" \
  --scale-rule-name servicebus-scaling \
  --scale-rule-type azure-servicebus \
  --scale-rule-metadata "queueName=orders" \
                        "namespace=sb-ecommerce" \
                        "messageCount=5" \
  --scale-rule-auth "connection=sb-connection"
```
Dla tematów zamień `queueName` na `topicName` i dodaj `subscriptionName`; każda subskrypcja ma własny licznik, więc konsumenci skalują się niezależnie. Parametr `--scale-rule-auth` mapuje nazwę sekretu na parametr uwierzytelniania scalera (`connection=sb-connection`).

## B3. Azure Storage Queue
_Źródło: [unit 3]_

Prostsza i tańsza alternatywa dla Service Bus (mniej funkcji, niższe limity przepustowości). Scaler liczy przybliżoną liczbę wiadomości. Metadane: `queueName`, `accountName`, `queueLength` (próg, podobnie do `messageCount`). Uwierzytelnianie: ciąg połączenia w sekrecie albo tożsamość zarządzana. Wybierz Storage Queue do prostego przetwarzania kolejki; gdy potrzebujesz sesji, dead-letter queue lub wiadomości zaplanowanych albo większej przepustowości, wybierz Service Bus.

## B4. Azure Event Hubs
_Źródło: [unit 3]_

Scaler Event Hubs jest dla strumieni o wysokiej przepustowości, przetwarzanych w **partycjach**. Monitoruje opóźnienie (lag) między najnowszym zdarzeniem w partycji a pozycją punktu kontrolnego (checkpoint) grupy konsumentów. Metadane: `consumerGroup`, `unprocessedEventThreshold` (liczba nieprzetworzonych zdarzeń na partycję), `checkpointStrategy` (zalecane `blobMetadata` przy punktach kontrolnych w Blob Storage).

**Pułapka egzaminacyjna:** w grupie konsumentów jedną partycję czyta jeden konsument naraz, więc replik ponad liczbę partycji nie ma sensu (32 partycje: `maxReplicas` powyżej 32 nic nie daje).

```yaml
# Code fragment - focus on Event Hubs scaler configuration
scale:
  minReplicas: 0
  maxReplicas: 32
  rules:
    - name: eventhubs-scaling
      custom:
        type: azure-eventhub
        metadata:
          consumerGroup: "$Default"
          unprocessedEventThreshold: "64"
          checkpointStrategy: "blobMetadata"
        auth:
          - secretRef: eh-connection
            triggerParameter: connection
```

## B5. Uwierzytelnianie reguł skalowania
_Źródło: [unit 3] + [docs: https://learn.microsoft.com/azure/container-apps/scale-app]_

Scalery niestandardowe muszą mieć dostęp do monitorowanych źródeł. Dwie metody:

| Metoda | Jak | Uwagi |
|---|---|---|
| **Sekrety** | sekret z ciągiem połączenia lub kluczem; mapowanie w `--scale-rule-auth` | działa zawsze, ale rotujesz ręcznie |
| **Tożsamość zarządzana** | tożsamość przypisana użytkownikowi + rola na zasobie (np. Azure Service Bus Data Receiver), flaga `--scale-rule-identity` | bez sekretów; zalecane na produkcji, gdy scaler to wspiera |

```azurecli
az containerapp create \
  --name queue-processor \
  --resource-group rg-ecommerce \
  --environment my-environment \
  --image myregistry.azurecr.io/queue-processor:v1 \
  --user-assigned <MANAGED_IDENTITY_RESOURCE_ID> \
  --min-replicas 0 \
  --max-replicas 20 \
  --scale-rule-name storage-queue-scaling \
  --scale-rule-type azure-queue \
  --scale-rule-metadata "accountName=stecommerce" \
                        "queueName=inventory-updates" \
                        "queueLength=10" \
  --scale-rule-identity <MANAGED_IDENTITY_RESOURCE_ID>
```
Dokumentacja dodaje: tożsamość zarządzana w regułach skalowania działa dla zasobów Azure (Queue Storage, Service Bus, Event Hubs) i **nie jest wspierana w portalu Azure** - użyj CLI lub ARM. [docs: https://learn.microsoft.com/azure/container-apps/scale-app]

## B6. Dobre praktyki
_Źródło: [unit 3]_

- **Tożsamość zarządzana** zamiast sekretów, z minimalnymi uprawnieniami RBAC.
- **Progi według mocy przerobowej:** zmierz przepustowość repliki; `messageCount` to docelowa głębokość kolejki na replikę, a **nie** współbieżność przetwarzania.
- **Skalowanie do zera** dla procesorów kolejek.
- **Obserwuj głębokość kolejki obok liczby replik** (Azure Monitor): gdy kolejka rośnie szybciej niż repliki, obniż próg; gdy repliki stoją bezczynne, podnieś.

---

# CZĘŚĆ C - SCALERY KEDA DLA NIESTANDARDOWYCH OBCIĄŻEŃ

## C1. Dostępne scalery
_Źródło: [unit 4]_

Container Apps obsługuje **dowolny scaler oparty na ScaledObject**: m.in. Apache Kafka, Redis Lists i Streams, Prometheus, Cron, PostgreSQL, MySQL, MongoDB, zewnętrzne API metryk. Wybierając scaler, sprawdź obsługiwane metody uwierzytelniania, wystawiane metryki i sposób przeliczania na repliki (dokumentacja KEDA scalers). Scalery dzielą się na utrzymywane przez Microsoft (Azure), społecznościowe (różna jakość dokumentacji) i **zewnętrzne** (osobne komponenty, wymagają dodatkowego wdrożenia poza wbudowaną konfiguracją).

## C2. Apache Kafka
_Źródło: [unit 4]_

Scaler Kafka skaluje według **opóźnienia grupy konsumentów** (consumer lag): różnica między najnowszym offsetem w partycji a zatwierdzonym offsetem grupy. Metadane: `bootstrapServers`, `consumerGroup`, `topic`, `lagThreshold` (opóźnienie na partycję; przy 100 i 500 zaległych wiadomościach wychodzi 5 replik). Uwierzytelnianie zwykle SASL (PLAIN, SCRAM) lub TLS, z poświadczeniami w sekretach. Tak jak w Event Hubs, liczba partycji ogranicza sensowną liczbę replik.

```yaml
# Code fragment - focus on Kafka scaler configuration
scale:
  minReplicas: 1
  maxReplicas: 50
  rules:
    - name: kafka-scaling
      custom:
        type: kafka
        metadata:
          bootstrapServers: "kafka-broker:9092"
          consumerGroup: "order-consumers"
          topic: "orders"
          lagThreshold: "100"
        auth:
          - secretRef: kafka-credentials
            triggerParameter: sasl
```

## C3. Redis
_Źródło: [unit 4]_

Dwa rodzaje scalerów: **Redis Lists** (długość listy przez `LLEN`; metadane `address`, `listName`, `listLength` jako próg; proste wzorce producent-konsument) i **Redis Streams** (liczba oczekujących wpisów (pending entries) dla grupy konsumentów, z uwzględnieniem wiadomości dostarczonych, ale niepotwierdzonych, więc lepiej radzi sobie z awariami). Obsługiwane są wdrożenia standardowe, Redis Cluster i Redis Sentinel; każdy wariant ma własny format połączenia.

## C4. Skalowanie według harmonogramu (cron)
_Źródło: [unit 4]_

Scaler **cron** działa według czasu, nie metryk: w oknie czasowym żąda stałej liczby replik, poza oknem nie wpływa na skalowanie. Metadane: `timezone`, `start`, `end` (wyrażenia cron) i `desiredReplicas`. Najlepiej łączyć z innymi regułami: cron daje bazę przed znanym szczytem, a HTTP lub zdarzenia obsługują nieoczekiwane skoki. Gdy aktywnych jest wiele scalerów, używana jest **najwyższa** żądana liczba replik.

```yaml
# Code fragment - focus on cron scaler configuration
scale:
  minReplicas: 0
  maxReplicas: 20
  rules:
    - name: business-hours
      custom:
        type: cron
        metadata:
          timezone: "America/New_York"
          start: "0 8 * * 1-5"
          end: "0 18 * * 1-5"
          desiredReplicas: "5"
    - name: http-scaling
      http:
        metadata:
          concurrentRequests: "50"
```
Poza godzinami pracy, gdy nie ma żądań, aplikacja może zejść do zera; w godzinach pracy działa co najmniej 5 replik niezależnie od ruchu.

## C5. Prometheus
_Źródło: [unit 4]_

Scaler Prometheus odpytuje serwer Prometheus i porównuje wynik z progiem. Metadane: `serverAddress`, `metricName`, `query` (wyrażenie PromQL zwracające liczbę) i `threshold`; wartość dzielona przez próg daje żądaną liczbę replik. Używaj, gdy metryki standardowe (współbieżność HTTP, głębokość kolejki) nie oddają obciążenia: np. aktywne sesje użytkowników lub oczekujące transakcje.

## C6. Konwersja specyfikacji KEDA na reguły Container Apps
_Źródło: [unit 4] + [docs: https://learn.microsoft.com/azure/container-apps/scale-app]_

1. **Typ** z `triggers[].type` -> `--scale-rule-type` albo `custom.type` w YAML.
2. **Metadane** z `triggers[].metadata` -> `--scale-rule-metadata` albo obiekt `metadata`.
3. **Uwierzytelnianie** z `TriggerAuthentication` -> sekrety Container Apps i `--scale-rule-auth` (albo tablica `auth`); alternatywnie `--scale-rule-identity`.
4. **Limity**: `--min-replicas` / `--max-replicas` odpowiadają `minReplicaCount` / `maxReplicaCount` w ScaledObject.

Różnice względem czystego KEDA: Container Apps **nie wspiera pełnego zasobu TriggerAuthentication** (sekrety wskazujesz bezpośrednio w regule), a zaawansowane funkcje (scalery zewnętrzne, własne interwały) mogą być niedostępne lub skonfigurowane inaczej. [unit 4]

## C7. Dobre praktyki
_Źródło: [unit 4]_

- **Zacznij od scalerów Azure** (Service Bus, Event Hubs, Storage Queue): wsparcie pierwszoplanowe.
- **Testuj w staging:** scalery niestandardowe mogą mieć nieoczekiwane odpytywanie i progi.
- **Łącz harmonogram i reakcję:** cron dla bazy, zdarzenia lub HTTP dla skoków.
- **Dokumentuj konfiguracje:** metadane scalera nie są samoopisujące się.

---

# CZĘŚĆ D - ZASOBY OBLICZENIOWE: WYDAJNOŚĆ I KOSZT

## D1. Alokacja CPU i pamięci
_Źródło: [unit 5] + [docs: https://learn.microsoft.com/azure/container-apps/containers]_

CPU podajesz w rdzeniach (także ułamkach), pamięć w GiB. Każda replika dostaje skonfigurowane zasoby, a platforma pilnuje limitów. Rozliczanie jest w **vCPU-sekundach** i **GiB-sekundach**. Przekroczenie limitu pamięci kończy się **zabiciem i restartem** repliki; przekroczenie CPU kończy się **throttlingiem** (spowolnieniem) bez zabijania. Czyli limit pamięci to twarda awaria, limit CPU to degradacja wydajności. [unit 5]

> **UWAGA - błąd w materiale Microsoftu** (unit 5): moduł pisze, że "memory must be at least twice the CPU value in GiB" (pamięć co najmniej dwa razy większa niż CPU) -> w planie Consumption dozwolone są tylko **ustalone pary** vCPU / pamięć, a pamięć to dokładnie 2 x vCPU (np. 0.25 / 0.5Gi, 0.5 / 1.0Gi, 1.0 / 2.0Gi, 4.0 / 8.0Gi), więc np. 0.5 rdzenia z 1.5Gi jest niedozwolone. Źródło: https://learn.microsoft.com/azure/container-apps/containers.

## D2. Konfiguracja kontenera i limity
_Źródło: [unit 5] + [docs: https://learn.microsoft.com/azure/container-apps/containers]_

Parametry `--cpu` i `--memory` podajesz przy tworzeniu lub aktualizacji. Domyślna alokacja to **0.25 rdzenia i 0.5 GiB** (dla lekkich aplikacji); w planie Consumption maksimum to **4 rdzenie i 8 GiB**. Łączna pojemność to zasoby repliki razy liczba replik (20 replik po 0.5 rdzenia = 10 rdzeni). [unit 5]

```azurecli
az containerapp create \
  --name order-api \
  --resource-group rg-ecommerce \
  --environment my-environment \
  --image myregistry.azurecr.io/order-api:v1 \
  --cpu 0.5 \
  --memory 1.0Gi \
  --min-replicas 2 \
  --max-replicas 20
```

> **UWAGA - błąd w materiale Microsoftu** (unit 5): moduł mówi, że limity dotyczą "per container, not per replica" i że każdy kontener w aplikacji (np. główny i sidecar) ma własną alokację -> dokumentacja mówi, że w planie Consumption **suma** CPU i pamięci wszystkich kontenerów aplikacji musi dawać jedną z dozwolonych kombinacji (czyli limit dotyczy łącznie całej repliki). Dodatkowo w środowisku **Consumption only** (legacy) maksimum to 2 rdzenie i 4Gi. Źródło: https://learn.microsoft.com/azure/container-apps/containers.

## D3. Profile obciążeń (workload profiles)
_Źródło: [unit 5] + [docs: https://learn.microsoft.com/azure/container-apps/workload-profiles-overview]_

| Typ środowiska | Co daje |
|---|---|
| **Consumption-only** (legacy) | model serverless, płacisz za zużyte zasoby |
| **Workload profiles** (domyślny) | profil **Consumption** (serverless, jak wyżej) plus profile **Dedicated** (zarezerwowane VM) |

Profile Dedicated dają stałą wydajność i większe alokacje; moduł twierdzi, że są **wymagane** do GPU i gdy potrzeba więcej niż 4 rdzenie lub 8 GiB na kontener (druga część jest zgodna z dokumentacją, pierwsza już nie - patrz uwaga niżej). Consumption jest zwykle opłacalny dla web API i przetwarzania w tle bez planowania pojemności; Dedicated wybierz dla stałych opóźnień, GPU, większych zasobów albo wymogów zgodności. > **UWAGA - błąd w materiale Microsoftu** (unit 5): "Dedicated profiles are required for GPU workloads" -> GPU są dostępne także w modelu serverless: profile **Consumption-GPU-NC24-A100** i **Consumption-GPU-NC8as-T4** (rozliczane per replika), obok profili Dedicated z GPU (NC24-A100 i większe). Źródło: https://learn.microsoft.com/azure/container-apps/workload-profiles-overview.

Dokumentacja: profil Consumption ma zakres 0.25-4 vCPU i 0.5-8 GiB (per replika), Dedicated (D4-D32, E4-E32) rozlicza się **per węzeł**. [docs: https://learn.microsoft.com/azure/container-apps/workload-profiles-overview]

## D4. Optymalizacja kosztu
_Źródło: [unit 5]_

Płacisz za aktywne repliki według zaalokowanych zasobów; repliki bezczynne mają niższą stawkę idle; przy skalowaniu do zera nie ma kosztu obliczeń. **Skalowanie do zera** to najskuteczniejsza optymalizacja dla obciążeń sporadycznych, w połączeniu z wyzwalaczami, które niezawodnie budzą repliki. Dobieraj rozmiar do rzeczywistego użycia: zacznij od domyślnych 0.25 / 0.5 GiB i zwiększaj tylko przy throttlingu, błędach braku pamięci (OOM) lub degradacji wydajności. Consumption zwykle jest tańszy dla zmiennego ruchu; Dedicated ma sens, gdy stałe wykorzystanie uzasadnia rezerwację.

## D5. Optymalizacja wydajności
_Źródło: [unit 5]_

Za mało zasobów to większe opóźnienia, timeouty i gorsze doświadczenie w szczycie. CPU dla pracy obliczeniowej (serializacja JSON, obrazy), pamięć dla cache'y, dużych ładunków i agregacji. **Minimalna liczba replik** eliminuje zimny start (opóźnienie pierwszego żądania po skalowaniu z zera). Profile Dedicated dają bardziej przewidywalną wydajność, bo zasoby nie są dzielone z innymi najemcami. Monitoruj użycie CPU i pamięci w Azure Monitor.

## D6. Balans limitów skalowania i zasobów
_Źródło: [unit 5]_

Pojemność = zasoby repliki x maksymalna liczba replik. Jeśli replika (0.5 rdzenia) obsługuje 100 żądań/s, a szczyt to 2000 żądań/s, potrzebujesz maksymalnie 20 replik (`--max-replicas 20`, `--cpu 0.5`). **Kompromis:** większe repliki to mniej skalowań, ale grubsza ziarnistość (możliwy nadmiar mocy między krokami); mniejsze repliki skalują precyzyjniej, ale generują więcej zdarzeń skalowania. Maksymalna liczba replik to **1000 na rewizję**. [unit 5]

## D7. Dobre praktyki
_Źródło: [unit 5]_

- Zacznij od domyślnych zasobów i skaluj na podstawie metryk.
- Consumption dla większości obciążeń; Dedicated tylko, gdy naprawdę potrzebne.
- Monitoruj rzeczywiste użycie: nadmiarowe kontenery marnują pieniądze, niedowymiarowane psują wydajność.
- Weź pod uwagę łączny koszt skalowania: małe kontenery skalują się częściej, co daje narzut i możliwy wpływ na żądania.

---

# CZĘŚĆ E - TRYBY REWIZJI I PODZIAŁ RUCHU

## E1. Rewizje a skalowanie
_Źródło: [unit 6] + [docs: https://learn.microsoft.com/azure/container-apps/revisions]_

Rewizja to niezmienna migawka wersji aplikacji (niemodyfikowalna; zmiana = nowa rewizja, co daje pewny cel wycofania). Zmiany **revision-scope** (sekcja `template`: obrazy, reguły skalowania, zmienne środowiskowe) tworzą rewizję; zmiany **application-scope** (sekretów, ingress, reguł podziału ruchu) nie. [unit 6] (szczegóły: `04-deploy-containers-azure-container-apps.md` §B4)

**Każda aktywna rewizja skaluje się niezależnie** według własnych reguł i swojego udziału w ruchu. Dwie rewizje z tymi samymi regułami i podziałem ruchu skalują się każda od swojej części ruchu, więc łączne zużycie może być większe niż jednej rewizji przy tym samym ruchu. Repliki są instancjami konkretnej rewizji. [unit 6]

## E2. Tryb Single (domyślny)
_Źródło: [unit 6]_

W trybie **Single** (domyślnym dla nowych aplikacji) aktywna jest jedna rewizja. Po wdrożeniu zmian platforma tworzy nową rewizję i czeka, aż będzie gotowa, zanim przeniesie ruch: **wdrożenie bez przestoju** (zero-downtime). Stara rewizja działa, dopóki nowa nie przejdzie testów zdrowia i nie osiągnie liczby replik poprzedniej; potem stara się dezaktywuje. Zaleta: prostota, brak ręcznych dezaktywacji. Wada: brak podziału ruchu i stopniowych wdrożeń; do testów na części ruchu potrzebny jest tryb Multiple.

## E3. Tryb Multiple
_Źródło: [unit 6] + [docs: https://learn.microsoft.com/cli/azure/containerapp?view=azure-cli-latest]_

W trybie **Multiple** wiele rewizji jest aktywnych naraz; każda może dostawać ruch i skalować się niezależnie, aż ją jawnie dezaktywujesz. Umożliwia wdrożenia canary, blue-green i testy A/B. Nowe wdrożenia nie dezaktywują poprzednich rewizji: Ty decydujesz, kiedy przenieść ruch i dezaktywować starą. Rewizje mają dwa stany: aktywna (przyjmuje ruch, skaluje się) i nieaktywna (nie zużywa zasobów, można ją reaktywować); platforma trzyma do 100 nieaktywnych, usuwając najstarsze.

```azurecli
az containerapp update \
  --name order-api \
  --resource-group rg-ecommerce \
  --revisions-mode multiple
```

> **UWAGA - błąd w materiale Microsoftu** (unit 6): przykład używa parametru `--revision-mode multiple` -> poprawna nazwa parametru w `az containerapp create`/`update` to `--revisions-mode` (wartości `labels`, `multiple`, `single`); alternatywa to `az containerapp revision set-mode --mode multiple`. Źródło: https://learn.microsoft.com/cli/azure/containerapp?view=azure-cli-latest (lokalny test pomocy CLI 2026-11-22: w `az containerapp update -h` jest tylko `--revisions-mode`).

## E4. Podział ruchu (traffic splitting)
_Źródło: [unit 6]_

Podział ruchu rozdziela żądania między aktywne rewizje według **wag w procentach, sumujących się do 100**. Rozkład jest losowy per żądanie, więc rzeczywiste proporcje mogą lekko odbiegać. Przykład: 80% na pierwszą wersję, 20% na drugą:

```azurecli
az containerapp ingress traffic set \
  --name order-api \
  --resource-group rg-ecommerce \
  --revision-weight order-api--v1=80 order-api--v2=20
```
Canary: zacznij od małego procentu (5-10%), obserwuj błędy i wydajność, potem zwiększaj; w razie problemów szybko cofnij ruch. Możesz też ustawić wagę dla `latest` zamiast konkretnej rewizji, wtedy nowe wdrożenia automatycznie dostają ten udział bez zmiany reguł (wygodne w potokach CD).

## E5. Etykiety rewizji (labels)
_Źródło: [unit 6]_

**Etykieta** to nazwany punkt końcowy, który kieruje wprost na konkretną rewizję, **niezależnie od podziału ruchu**; każda etykieta ma własny adres URL. Zastosowania: canary (etykieta na nowej rewizji i adres dla testerów przed dodaniem jej do podziału) i blue-green (etykiety "blue" i "green", a produkcję przełączasz **przesunięciem etykiety**, nie zmianą wag). Zasady nazw: zaczyna się od litery, tylko małe litery, cyfry i `-`, bez `--`, maks. 64 znaki. Etykieta wskazuje jedną rewizję naraz, ale można ją przenosić (przeniesienie jest atomowe). Główny adres aplikacji podlega podziałowi ruchu, adres etykiety zawsze trafia do rewizji z etykietą; oba mechanizmy można łączyć. Polecenia CLI: `az containerapp revision label add | remove | swap` (lokalny test pomocy CLI [autor: test 2026-11-22]).

## E6. Skalowanie przy wielu rewizjach
_Źródło: [unit 6]_

Dwie rewizje po 50% ruchu mogą razem zużywać więcej zasobów niż jedna z 100%, bo każda ma własne minimum replik i skaluje się od własnych metryk. Przykład: dwie rewizje z `minReplicas: 1` i podziałem 50/50 mają łącznie co najmniej dwie repliki; jeśli reguły dałyby trzecią replikę przy pełnym ruchu, każda rewizja widzi tylko połowę ruchu i może nie uruchomić skalowania. Planuj przejścia: wdróż, sprawdź na 10%, krótko 50%, potem 100% i **szybko dezaktywuj** starą rewizję. Monitoruj każdą rewizję osobno (liczba replik, użycie zasobów, błędy).

## E7. Dobre praktyki
_Źródło: [unit 6]_

- **Single dla prostych wdrożeń** (zero-downtime bez konfiguracji).
- **Multiple dla canary** z planem procentów i walidacji.
- **Dezaktywuj stare rewizje szybko:** aktywne rewizje skalują się i zużywają zasoby; nieaktywne można reaktywować przy wycofaniu.
- **Najpierw etykiety, potem podział ruchu:** walidacja nowej rewizji bez wpływu na ruch produkcyjny.

---

# CZĘŚĆ F - ĆWICZENIA

Przeczytaj części A-E. Każde ćwiczenie ma ten sam układ pól (przewodnik stylu §5). Ćwiczenia I1 i I2 tworzą płatne zasoby w chmurze: robisz je w jednej grupie zasobów i wykonujesz **Sprzątanie** po I2 (I2 kontynuuje I1). Kod aplikacji i SDK jest w Pythonie; odpowiednik C# jest w ćwiczeniu z kodem (I3).

> Wersje sprawdzone 2026-11-22 (źródło: pypi.org, api.nuget.org, `az version`, testy lokalne): Python 3.12 (`python:3.12-slim`), Flask 3.1.3, gunicorn 27.0.1, requests 2.34.2, azure-mgmt-appcontainers 5.1.0, azure-identity 1.27.0; C#: .NET SDK 10.0, Azure.ResourceManager.AppContainers 1.6.0, Azure.Identity 1.21.0; Azure CLI 2.90.0, rozszerzenie containerapp 1.3.0b6 (beta). Starter laboratorium Microsoftu pinuje starsze flask==3.0.0 i gunicorn==22.0.0. Przed użyciem sprawdź PyPI i NuGet.

## I1. Skalowanie HTTP pod obciążeniem i konfiguracja w YAML (lab modułu)
(rozwinięcie jednostki 7; scenariusz: [unit 7])

**Gdzie / Plik:** terminal z Azure CLI (Bash); pliki `api/main.py`, `api/requirements.txt`, `api/Dockerfile`, `load.py`.

> Uwaga o labie: lab Microsoftu używa skryptu `azdeploy.py`, startowej aplikacji z endpointem `/agent/process` oraz lokalnego dashboardu Flask w katalogu `client` do generowania obciążenia; tutaj zamiast dashboardu masz skrypt `load.py`. Lab wymaga planu Pay-As-You-Go lub innego płatnego (zadania ACR Tasks są tymczasowo wstrzymane dla darmowych kredytów) [unit 7].

`api/main.py` (symulacja pracy modelu: endpoint z opóźnieniem; sprawdzony lokalnie):
```python
import os
import time

from flask import Flask, jsonify, request

app = Flask(__name__)

DEFAULT_DELAY_MS = int(os.getenv("AGENT_DEFAULT_DELAY_MS", "500"))


@app.get("/health")
def health():
    return jsonify(status="healthy")


@app.get("/")
def root():
    return jsonify(service="mock-agent-api", default_delay_ms=DEFAULT_DELAY_MS)


@app.post("/agent/process")
def agent_process():
    payload = request.get_json(silent=True) or {}
    try:
        delay_ms = int(payload.get("delayMs", DEFAULT_DELAY_MS))
    except (TypeError, ValueError):
        delay_ms = DEFAULT_DELAY_MS
    delay_ms = max(0, min(delay_ms, 30_000))   # symulacja pracy modelu
    time.sleep(delay_ms / 1000.0)
    return jsonify(status="done", delay_ms=delay_ms)
```
`api/requirements.txt`:
```
flask==3.1.3
gunicorn==27.0.1
```
`api/Dockerfile` (jeden worker z 32 wątkami, żeby jedna replika mogła trzymać wiele równoległych żądań):
```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY main.py .
ENV PORT=8000
EXPOSE 8000
CMD ["sh", "-c", "gunicorn --bind 0.0.0.0:${PORT:-8000} --workers 1 --threads 32 --access-logfile - main:app"]
```
`load.py` (generator równoległych żądań, sprawdzony lokalnie: 8 wątków, 3 s, 80 odpowiedzi OK):
```python
import os
import sys
import time
from concurrent.futures import ThreadPoolExecutor

import requests

base_url = os.environ["APP_URL"].rstrip("/")          # np. https://<fqdn>
concurrency = int(sys.argv[1]) if len(sys.argv) > 1 else 25
duration_s = int(sys.argv[2]) if len(sys.argv) > 2 else 60
delay_ms = int(sys.argv[3]) if len(sys.argv) > 3 else 1000

stop_at = time.time() + duration_s
sent = ok = failed = 0


def worker(_: int) -> tuple[int, int]:
    done_ok = done_fail = 0
    with requests.Session() as session:               # (1) jedna sesja = jedno polaczenie na watek
        while time.time() < stop_at:
            try:
                r = session.post(f"{base_url}/agent/process", json={"delayMs": delay_ms}, timeout=40)
                if r.ok:
                    done_ok += 1
                else:
                    done_fail += 1
            except requests.RequestException:
                done_fail += 1
    return done_ok, done_fail


with ThreadPoolExecutor(max_workers=concurrency) as pool:   # (2) tyle rownoleglych zadan, ile watkow
    for o, f in pool.map(worker, range(concurrency)):
        ok += o
        failed += f

print(f"concurrency={concurrency} duration={duration_s}s ok={ok} failed={failed}")
```
Polecenia:
```bash
RG=rg-aca-scale-i1; LOC=westeurope
ACR_NAME=acrscale$RANDOM$RANDOM; APP=ai-agent
pip install requests==2.34.2
az extension add --name containerapp --upgrade
az provider register --namespace Microsoft.App
az provider register --namespace Microsoft.OperationalInsights
az provider register --namespace Microsoft.ContainerRegistry

az group create --name $RG --location $LOC
az acr create --resource-group $RG --name $ACR_NAME --sku Basic --admin-enabled false
az acr build --registry $ACR_NAME --image ai-agent:v1 --file api/Dockerfile api
az containerapp env create --name aca-env-scale --resource-group $RG --location $LOC

az containerapp create --name $APP --resource-group $RG --environment aca-env-scale \
    --image $ACR_NAME.azurecr.io/ai-agent:v1 --ingress external --target-port 8000 \
    --registry-server $ACR_NAME.azurecr.io --registry-identity system \
    --revision-suffix v1 \
    --min-replicas 0 --max-replicas 10 \
    --scale-rule-name http-scaling --scale-rule-type http --scale-rule-http-concurrency 10   # (1)

az containerapp show -n $APP -g $RG --query "properties.template.scale"                      # (2)
export APP_URL=https://$(az containerapp show -n $APP -g $RG --query properties.configuration.ingress.fqdn -o tsv)

python load.py 40 90 1000 &                                                                  # (3)
for i in 1 2 3 4 5 6 7 8 9; do                                                               # (4)
  date +%T; az containerapp replica list -n $APP -g $RG --query "length(@)"; sleep 10
done
wait
```
Objaśnienie: (1) reguła HTTP z progiem 10 równoległych żądań na replikę, minimum 0, maksimum 10; (2) odczyt sekcji `scale` szablonu (lab: szukaj reguły `http-scaling`, `minReplicas` 0 i `maxReplicas` 10) [unit 7]; (3) 40 równoległych wątków przez 90 s, każde żądanie trwa ok. 1 s; (4) liczba replik co 10 s (`length(@)` to długość listy w JMESPath).

Teraz konfiguracja w YAML (lab: eksport, edycja, zastosowanie) [unit 7]:
```bash
az containerapp show -n $APP -g $RG --output yaml > app-config.yaml
```
W `app-config.yaml` w sekcji `properties.template.scale` ustaw:
```yaml
scale:
  cooldownPeriod: 200
  maxReplicas: 5
  minReplicas: 1
  pollingInterval: 30
```
```bash
az containerapp update -n $APP -g $RG --yaml app-config.yaml
az containerapp show -n $APP -g $RG --query "properties.template.scale"
```
Typowe problemy z labu [unit 7]: brak skalowania - sprawdź regułę (`az containerapp show --query "properties.template.scale"`), upewnij się, że żądania są równoległe (opóźnienie `delayMs` 500-1500 ms sprawia, że się nakładają) i obejrzyj logi systemowe `az containerapp logs show --type system --tail 50`; aktualizacja YAML kończy się błędem - sprawdź wcięcia, a pola tylko do odczytu (`id`, `systemData`, `type`) usuń z pliku.

**Koszt:** ACR Basic naliczany dziennie (stawka: Niesprawdzone - zweryfikuj w https://azure.microsoft.com/pricing/details/container-registry/); Container Apps w planie Consumption: pierwsze 180 000 vCPU-s, 360 000 GiB-s i 2 mln żądań HTTP miesięcznie są bezpłatne, replika skalowana do zera nie kosztuje, repliki bezczynne mają stawkę idle [docs: https://learn.microsoft.com/azure/container-apps/billing]. Test obciążeniowy uruchamia do 10 małych replik na kilka minut (domyślnie 0.25 vCPU i 0.5 GiB) - to ułamek darmowego progu. Log Analytics rozlicza ilość logów (pomijalna). Po ćwiczeniach wykonaj Sprzątanie w I2.

**Zadania:** (a) wdróż aplikację i obserwuj liczbę replik w czasie testu obciążeniowego; (b) po zakończeniu obciążenia obserwuj, jak długo repliki zostają przy minimum 0 (cool-down); (c) **zepsuj to:** ustaw wysoki próg `az containerapp update -n $APP -g $RG --scale-rule-name http-scaling --scale-rule-type http --scale-rule-http-concurrency 1000`, uruchom obciążenie ponownie i porównaj liczbę replik; (d) zastosuj YAML ze zmianą `cooldownPeriod`, `minReplicas`, `maxReplicas` i sprawdź, czy powstała nowa rewizja (`az containerapp revision list -n $APP -g $RG -o table`).

**Sprawdź:** (a) liczba replik rośnie ponad 1 w trakcie obciążenia (przy 40 równoległych żądaniach i progu 10 wzór `ceil(40/10)` daje rząd 4 replik; dokładna liczba zależy od uśredniania 15 s) [docs: https://learn.microsoft.com/azure/container-apps/scale-app] [autor: niezweryfikowane]; (b) po ustaniu obciążenia zejście do zera następuje po okresie cool-down (domyślnie 300 s) [unit 2]; (c) przy progu 1000 pojedyncza replika obsługuje cały ruch, więc nie ma skalowania w górę (uwaga: 40 żądań to mniej niż 1000) [autor: niezweryfikowane]; (d) zmiana sekcji `scale` jest zmianą w `template`, więc powstaje nowa rewizja [unit 7] [docs: https://learn.microsoft.com/azure/container-apps/revisions].

**Sprzątanie:** zasoby (grupa `rg-aca-scale-i1`) są potrzebne jeszcze w I2 i I3, więc usuń je dopiero po nich (Sprzątanie z I2). Jeśli kończysz na I1, wykonaj od razu: `az group delete --name $RG --yes --no-wait`, a potem `az group exists --name $RG` (po kilku minutach powinno wypisać `false`).

## I2. Tryb Multiple, podział ruchu 80/20 i etykiety
(rozwinięcie sekcji E3-E5; kontynuacja I1: ta sama grupa `rg-aca-scale-i1`, aplikacja `ai-agent`, zmienne `RG`, `APP` i `APP_URL` z I1)

**Gdzie / Plik:** terminal z Azure CLI (Bash) w tej samej sesji co I1.

```bash
az containerapp revision set-mode -n $APP -g $RG --mode multiple                                   # (1)

az containerapp update -n $APP -g $RG --revision-suffix v2 \
    --set-env-vars AGENT_DEFAULT_DELAY_MS=111                                                      # (2)
az containerapp revision list -n $APP -g $RG -o table

az containerapp ingress traffic set -n $APP -g $RG \
    --revision-weight $APP--v1=80 $APP--v2=20                                                      # (3)
az containerapp ingress traffic show -n $APP -g $RG

for i in $(seq 1 20); do curl -s $APP_URL/ ; echo; done                                            # (4)

az containerapp revision label add -n $APP -g $RG --label green --revision $APP--v2               # (5)
az containerapp ingress traffic set -n $APP -g $RG --revision-weight $APP--v1=60 $APP--v2=60       # (6)
```
Objaśnienie: (1) przełączenie trybu na Multiple (równoważne `az containerapp update --revisions-mode multiple`); (2) nowa rewizja `v2` ze zmienioną zmienną: odpowiedź `/` ma wtedy `default_delay_ms` równe 111 zamiast 500; (3) podział 80/20; (4) 20 wywołań pokazuje, która rewizja odpowiedziała (po wartości `default_delay_ms`); (5) etykieta `green` na `v2` daje osobny adres URL; (6) **zepsuj to:** wagi nie sumują się do 100.

**Koszt:** jak w I1 (te same zasoby); w trybie Multiple obie rewizje mogą mieć działające repliki, więc krótko płacisz za obie.

**Zadania:** (a) przełącz tryb i wdróż `v2`; (b) ustaw podział 80/20 i wykonaj 20 wywołań `/`; (c) dodaj etykietę `green` i wywołaj jej adres URL (adres etykiety odczytasz z `az containerapp revision show -n $APP -g $RG --revision $APP--v2 --query properties.fqdn -o tsv` lub z `ingress traffic show`); (d) **zepsuj to:** ustaw wagi 60 i 60.

**Sprawdź:** (a) `revision list` pokazuje `ai-agent--v1` i `ai-agent--v2` jako aktywne [unit 6]; (b) około 16 z 20 odpowiedzi ma `default_delay_ms` 500, a około 4 ma 111 (rozkład losowy, więc liczby mogą odbiegać) [unit 6]; (c) adres z etykietą zawsze zwraca 111 niezależnie od wag [unit 6]; (d) polecenie kończy się błędem, bo wagi nie sumują się do 100 (dokładny komunikat zależy od wersji CLI) [autor: niezweryfikowane].

**Sprzątanie:**
```bash
az group delete --name $RG --yes --no-wait
az group exists --name $RG      # po kilku minutach powinno wypisać false
```
Usuwa cały zestaw z I1 i I2 (ACR, środowisko, Log Analytics, aplikację). Wykonaj dopiero po I3.

## I3. Liczba replik przez SDK i generator obciążenia: Python i odpowiednik C# (.NET)
(rozwinięcie sekcji A5, E6; wymaga aplikacji z I1 przed jej Sprzątaniem w I2)

**Gdzie / Plik:** `watch.py` (Python; generator `load.py` jest w I1) oraz `Program.cs` (C#); `az login`.

```bash
pip install azure-mgmt-appcontainers==5.1.0 azure-identity==1.27.0
export AZURE_SUBSCRIPTION_ID=$(az account show --query id -o tsv)
export RESOURCE_GROUP=rg-aca-scale-i1 CONTAINER_APP_NAME=ai-agent
```
`watch.py`:
```python
import os
import time

from azure.identity import DefaultAzureCredential
from azure.mgmt.appcontainers import ContainerAppsAPIClient

subscription_id = os.environ["AZURE_SUBSCRIPTION_ID"]
resource_group = os.environ["RESOURCE_GROUP"]
app_name = os.environ["CONTAINER_APP_NAME"]

client = ContainerAppsAPIClient(DefaultAzureCredential(), subscription_id)

for _ in range(30):                                    # (1) 30 probek co 10 s = 5 minut
    stamp = time.strftime("%H:%M:%S")
    for rev in client.container_apps_revisions.list_revisions(resource_group, app_name):
        p = rev.properties
        if p.active:                                   # (2) tylko aktywne rewizje
            print(stamp, rev.name, "replicas =", p.replicas, "traffic =", p.traffic_weight)
    time.sleep(10)
```
Objaśnienie: (1) pętla próbkuje liczbę replik każdej aktywnej rewizji (pole `replicas` w `RevisionProperties`); (2) po I2 zobaczysz osobne liczby dla `v1` i `v2`, co ilustruje niezależne skalowanie rewizji (E6). Uruchom `python watch.py` w jednym terminalu, a `python load.py 40 90 1000` (z `APP_URL`) w drugim. Skrypt skompilowano (`py_compile`), a pola `replicas`, `active`, `traffic_weight` sprawdzono w modelu `RevisionProperties`; nie uruchamiano go na prawdziwej subskrypcji [autor: test 2026-11-22] [docs: https://learn.microsoft.com/python/api/overview/azure/mgmt-appcontainers-readme?view=azure-python].

**Odpowiednik C# (.NET)**

Różnice: równoległość robisz zadaniami asynchronicznymi (`Task.WhenAll` zamiast `ThreadPoolExecutor`), licznikami bezpiecznymi wątkowo (`Interlocked.Increment`), a klient HTTP to `HttpClient` z `PostAsJsonAsync`. Liczba replik rewizji to `rev.Data.Replicas`, a aktywność `IsActive`. Jeden plik obsługuje oba tryby: `dotnet run -- load 40 90 1000` (z `APP_URL`) i `dotnet run -- watch`. Projekt: `dotnet new console -f net10.0`, `dotnet add package Azure.ResourceManager.AppContainers --version 1.6.0`, `dotnet add package Azure.Identity --version 1.21.0`.
```csharp
using System.Net.Http.Json;
using Azure.Identity;
using Azure.ResourceManager;
using Azure.ResourceManager.AppContainers;

// Uzycie: dotnet run -- load  <concurrency> <seconds> <delayMs>   |   dotnet run -- watch
string mode = args.Length > 0 ? args[0] : "watch";

if (mode == "load")
{
    string baseUrl = Environment.GetEnvironmentVariable("APP_URL")!.TrimEnd('/');
    int concurrency = args.Length > 1 ? int.Parse(args[1]) : 25;
    int seconds = args.Length > 2 ? int.Parse(args[2]) : 60;
    int delayMs = args.Length > 3 ? int.Parse(args[3]) : 1000;

    using var http = new HttpClient { Timeout = TimeSpan.FromSeconds(40) };
    var stopAt = DateTime.UtcNow.AddSeconds(seconds);
    int ok = 0, failed = 0;

    var tasks = Enumerable.Range(0, concurrency).Select(async _ =>
    {
        while (DateTime.UtcNow < stopAt)
        {
            try
            {
                var r = await http.PostAsJsonAsync($"{baseUrl}/agent/process", new { delayMs });
                if (r.IsSuccessStatusCode) Interlocked.Increment(ref ok);
                else Interlocked.Increment(ref failed);
            }
            catch (HttpRequestException) { Interlocked.Increment(ref failed); }
            catch (TaskCanceledException) { Interlocked.Increment(ref failed); }
        }
    });
    await Task.WhenAll(tasks);
    Console.WriteLine($"concurrency={concurrency} duration={seconds}s ok={ok} failed={failed}");
}
else
{
    string sub = Environment.GetEnvironmentVariable("AZURE_SUBSCRIPTION_ID")!;
    string rg = Environment.GetEnvironmentVariable("RESOURCE_GROUP")!;
    string appName = Environment.GetEnvironmentVariable("CONTAINER_APP_NAME")!;

    var arm = new ArmClient(new DefaultAzureCredential());
    ContainerAppResource app = arm.GetContainerAppResource(
        ContainerAppResource.CreateResourceIdentifier(sub, rg, appName));

    for (int i = 0; i < 30; i++)
    {
        await foreach (ContainerAppRevisionResource rev in app.GetContainerAppRevisions().GetAllAsync())
            if (rev.Data.IsActive == true)
                Console.WriteLine($"{DateTime.Now:HH:mm:ss} {rev.Data.Name} replicas={rev.Data.Replicas} traffic={rev.Data.TrafficWeight}");
        await Task.Delay(TimeSpan.FromSeconds(10));
    }
}
```
[docs: https://learn.microsoft.com/dotnet/api/overview/azure/resourcemanager.appcontainers-readme?view=azure-dotnet] [autor: test 2026-11-22] (kod skompilowano na .NET SDK 10.0.401 z pakietami jak wyżej; nie uruchamiano go na prawdziwej subskrypcji ani wobec działającej aplikacji).

**Koszt:** brak kosztu własnego (odczyt metadanych); generator obciążenia generuje ruch do aplikacji z I1 (koszt jak tam).

**Zadania:** (a) uruchom `watch.py` podczas `load.py` i porównaj z pętlą CLI z I1; (b) **zepsuj to:** uruchom `load.py` z `APP_URL` wskazującym nieistniejący adres (np. `https://nie-ma.example.invalid`) i odczytaj wynik; (c) uruchom wersję C# (`load` i `watch`) i porównaj.

**Sprawdź:** (a) kolumna `replicas` rośnie w trakcie obciążenia i maleje po jego ustaniu (zejście do zera po cool-down); (b) wszystkie żądania kończą się niepowodzeniem: `failed` rośnie, `ok=0` (obsługa wyjątku w kodzie nie przerywa programu) [autor: niezweryfikowane]; (c) te same wartości co w Pythonie, z formatem `replicas=...`.

**Sprzątanie:** brak (skrypty tylko czytają metadane i generują ruch); po I3 wykonaj Sprzątanie z I2.

---

# CZĘŚĆ G - QUIZ Z MODUŁU MICROSOFTU (4 pytania) + wyjaśnienia

Learn nie pokazuje poprawnych odpowiedzi w ocenie modułu, więc odpowiedzi są wnioskowane z treści jednostek; przy każdym pytaniu jest krótki znacznik.

## G1. Przetwarzanie wiadomości z kolejki Service Bus i skalowanie do zera
-> *`--min-replicas 0` z regułą `azure-servicebus`.* Reguła HTTP nie widzi kolejki, a reguła CPU nie potrafi skalować do zera. *(odpowiedź wnioskowana: unit 3, sekcja "Configure Azure Service Bus scaling"; unit 2, sekcja "Configure CPU and memory scale rules")*

## G2. Pięć replik przed szczytem o 8:00 i skalowanie do zera nocą
-> *Reguła cron połączona z regułą HTTP.* Minimum pięciu replik wyklucza zero w nocy, a niski próg CPU nie gwarantuje pięciu replik o określonej godzinie i nie skaluje do zera. *(odpowiedź wnioskowana: unit 4, sekcja "Configure cron-based scaling")*

## G3. 10% ruchu na nową wersję przed pełnym wdrożeniem
-> *Włączyć tryb Multiple i skonfigurować podział ruchu wagami rewizji.* Tryb Single zapewnia tylko wdrożenie bez przestoju, a reguła cron nie przesuwa ruchu między wersjami. *(odpowiedź wnioskowana: unit 6, sekcje "Configure multiple revision mode" i "Implement traffic splitting")*

## G4. Zalecana metoda uwierzytelniania reguł KEDA na produkcji
-> *Tożsamość zarządzana (`--scale-rule-identity`).* Ciągi połączeń w sekretach działają, ale wymagają rotacji; dostęp anonimowy odpada. *(odpowiedź wnioskowana: unit 3, sekcja "Authenticate scale rules" i "Best practices")*

---

# CZĘŚĆ H - PYTANIA KONTROLNE (odpowiedz bez zaglądania)

## H1. Jakie są domyślne limity skalowania i co się stanie bez ingress i bez reguł?
<details><summary>Odpowiedź</summary>

Z włączonym ingress i bez własnych reguł: domyślna reguła HTTP, od 0 do 10 replik. Bez ingress, bez minimum i bez własnej reguły aplikacja skaluje się do zera i nie może się uruchomić ponownie.
</details>

## H2. Który typ reguły nie potrafi skalować do zera i dlaczego?
<details><summary>Odpowiedź</summary>

Reguły CPU i pamięci: potrzebują co najmniej jednej działającej repliki, żeby zmierzyć wykorzystanie. HTTP, TCP i zdarzeniowe skalują do zera.
</details>

## H3. Podaj wartości zachowania skalowania.
<details><summary>Odpowiedź</summary>

Polling interval 30 s (nie dotyczy HTTP/TCP), cool-down 300 s (tylko przy schodzeniu z ostatniej repliki do zera), scale-up stabilization 0 s, scale-down stabilization 300 s, kroki w górę 1, 4, 8, 16, 32..., w dół 100% naraz.
</details>

## H4. Ile replik zażąda scaler Service Bus przy 50 wiadomościach i `messageCount` 5? Jaki jest ogólny wzór?
<details><summary>Odpowiedź</summary>

10 replik. Wzór z dokumentacji: `desiredReplicas = ceil(currentMetricValue / targetMetricValue)`.
</details>

## H5. Dlaczego `maxReplicas` powyżej liczby partycji Event Hubs nic nie daje?
<details><summary>Odpowiedź</summary>

W grupie konsumentów jedną partycję czyta jeden konsument naraz, więc replik ponad liczbę partycji nie da się wykorzystać (to samo dotyczy Kafka).
</details>

## H6. Jak uwierzytelnić regułę skalowania bez sekretów?
<details><summary>Odpowiedź</summary>

Tożsamością zarządzaną: `--user-assigned <id>` na aplikacji, rola na zasobie (np. Azure Service Bus Data Receiver) i `--scale-rule-identity <id>`. Nie jest to wspierane w portalu, tylko CLI/ARM.
</details>

## H7. Jakie parametry CLI i pary CPU/pamięć są poprawne w planie Consumption?
<details><summary>Odpowiedź</summary>

`--cpu` i `--memory`; dozwolone tylko ustalone kombinacje (pamięć = 2 x vCPU: 0.25/0.5Gi ... 4.0/8.0Gi), liczone dla sumy kontenerów repliki; domyślnie 0.25 / 0.5 GiB.
</details>

## H8. Czym różnią się tryby Single i Multiple oraz jaki parametr je ustawia?
<details><summary>Odpowiedź</summary>

Single (domyślny): jedna aktywna rewizja, wdrożenie bez przestoju, brak podziału ruchu. Multiple: wiele aktywnych rewizji, podział ruchu, canary i blue-green. Parametr: `--revisions-mode` (albo `az containerapp revision set-mode --mode`).
</details>

## H9. Jak działają etykiety względem podziału ruchu?
<details><summary>Odpowiedź</summary>

Etykieta daje osobny adres URL, który zawsze kieruje do swojej rewizji niezależnie od wag. Główny adres aplikacji podlega podziałowi ruchu. Etykieta wskazuje jedną rewizję naraz, a przeniesienie jest atomowe.
</details>

---

# CZĘŚĆ J - CO DALEJ

## J1. Następny krok i pokrycie egzaminu
_Źródło: [moduł kursu: dynamic-sessions-azure-container-apps]_

Dalej: moduł 07 (dynamic sessions: bezpieczne wykonywanie kodu generowanego przez AI). Wróć do części B i C, gdy uczysz się scalerów (Service Bus, Event Hubs, Kafka, cron), a do E przy zadaniach o canary i etykietach.

**Pokrycie egzaminu** (wiersze z `sources/_exam/skill-map.md`, stan study guide z 2026-10-30):

| Umiejętność (study guide) | Domena | Status | Gdzie |
|---|---|---|---|
| Implement event-driven scaling by using KEDA in Container Apps | 1 | uczone tutaj | `06-scale-containers-azure-container-apps.md` §A1-§C7 |
| Deploy applications to Azure Container Apps, including environment configuration and revision management | 1 | uczone w modułach 04 i 05 (tu: tryby rewizji, podział ruchu, §E1-§E7) | `04-deploy-containers-azure-container-apps.md`; `05-manage-containers-azure-container-apps.md` |
| Queue and process back-end operations by using Azure Service Bus, including dead-letter queue handling, messages, topics, and subscriptions | 3 | tu tylko wzmianka (scaler Service Bus, §B2); właściciel: moduł kursu Service Bus | https://learn.microsoft.com/en-us/training/modules/queue-process-operations-service-bus/ |

## J2. Uczciwie o egzaminie
_Źródło: [autor]_

Ten moduł pokrywa wszystkie typy reguł skalowania, scalery KEDA dla usług Azure i dla Kafka, Redis, cron i Prometheus, uwierzytelnianie reguł, dobór zasobów oraz tryby rewizji z podziałem ruchu. Nie uczy samych usług, z których czytają scalery (Service Bus, Event Hubs, Storage) ani metod ich użycia w kodzie. Quiz ma tylko 4 pytania, więc egzamin będzie głębszy: ćwicz wartości domyślne (0-10 replik, 10 żądań na replikę, 30 s, 300 s), brak skalowania do zera dla CPU i pamięci, limit partycji oraz poprawne nazwy parametrów (`--revisions-mode`, `--scale-rule-identity`).
