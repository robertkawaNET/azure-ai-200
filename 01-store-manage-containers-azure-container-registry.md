# Moduł 01: Rejestr kontenerów w Azure (Azure Container Registry, ACR Tasks, tagowanie, wersjonowanie)

Moduł Microsoft Learn: **Store and manage containers in Azure Container Registry** - https://learn.microsoft.com/en-us/training/modules/store-manage-containers-azure-container-registry/
Ścieżka: Implement container application hosting on Azure (moduł 1 z 3), kurs AI-200T00, egzamin **AI-200** (Azure AI Cloud Developer Associate), domena 1 "Develop containerized solutions on Azure".
Cel: po tym module rozumiesz, jak ACR organizuje obrazy (rejestr, repozytorium, artefakt, tag, digest), umiesz zbudować obraz w chmurze przez ACR Tasks, wybrać strategię tagowania i zabezpieczyć oraz posprzątać obrazy.
Czas: ok. 4-6 h (czytanie + cztery ćwiczenia). Ćwiczenia robisz w Azure Cloud Shell (Bash) albo lokalnie z Azure CLI 2.x. Potrzebujesz subskrypcji **płatnej** (Pay-As-You-Go lub innej): uruchomienia ACR Tasks są tymczasowo wstrzymane dla darmowych kredytów Azure. Koszt: ćwiczenia tworzą rejestr ACR w poziomie Basic (płatny); szczegóły i sprzątanie w części D.

Język kodu: egzamin AI-200 zakłada programowanie w **Pythonie**, więc kod aplikacji i SDK jest podany w Pythonie. Przy ćwiczeniach z kodem aplikacji lub SDK dodano blok **Odpowiednik C# (.NET)** z opisem różnic. Polecenia `az` (Azure CLI), Dockerfile i teoria są wspólne dla obu języków i nie są powtarzane.

Znaczniki źródeł (linia `_Źródło: ..._` pod nagłówkiem):
- `[unit N]` - jednostka (unit) modułu Microsoft Learn; N to prefiks pliku w `sources/store-manage-containers-azure-container-registry/` (`5-lab` = README laboratorium, `5-starter` = pliki startowe laboratorium).
- `[moduł kursu: <slug>]` - inny moduł tego samego kursu.
- `[docs: URL]` - oficjalna dokumentacja produktu.
- `[autor]` - dodane przez autora tutoriala; `[autor: test <data>]` - sprawdzone testem w danym dniu; `[autor: niezweryfikowane]` - jeszcze nie sprawdzone w Azure.

W tekście "moduł N" znaczy plik tutoriala `0N-*.md` (np. moduł 02 = `02-deploy-containers-azure-app-service.md`, o ile już istnieje).

---

# CZĘŚĆ A - REJESTR: CO TO JEST I JAK ORGANIZUJE OBRAZY

## A1. Czym jest Azure Container Registry i jakie ma poziomy
_Źródło: [unit 1] + [unit 2] + [docs: https://learn.microsoft.com/en-us/azure/container-registry/container-registry-skus]_

**Obraz kontenera** (container image) to zapakowana aplikacja razem z jej zależnościami. Gdzieś trzeba taki obraz przechowywać, żeby serwery mogły go pobrać. Takie miejsce to **rejestr** (registry).

**Azure Container Registry (ACR)** to zarządzany, prywatny rejestr obrazów w stylu Dockera. W odróżnieniu od publicznego Docker Hub (publiczny rejestr obrazów) dajesz dostęp tylko wybranym osobom i usługom. ACR łączy się z Azure Kubernetes Service (AKS = zarządzany Kubernetes, system do uruchamiania wielu kontenerów w klastrze), Azure Container Apps (uruchamianie kontenerów bez zarządzania klastrem) i Azure App Service (hosting aplikacji webowych).

Scenariusz z modułu: zespół buduje obrazy (API do wnioskowania modelu AI, serwis wstępnego przetwarzania, kontener pomocniczy do monitoringu) na laptopach i wypycha je ręcznie. Efekt: różne buildy, niejasne wersje, brak możliwości cofnięcia. Rozwiązanie: jeden centralny rejestr, buildy w chmurze, spójne tagowanie.

Co daje ACR dla aplikacji AI:

| Możliwość | Znaczenie |
|---|---|
| Prywatne przechowywanie | Obrazy zostają w Twoim środowisku Azure |
| Geo-replikacja (geo-replication) | Kopie obrazów w wielu regionach: szybsze pobieranie i mniejsze opóźnienia |
| Integracja | AKS, Container Apps, App Service pobierają obrazy bezpośrednio |
| Formaty | Obrazy Dockera, wykresy Helm (Helm chart = paczka konfiguracji dla Kubernetes) i artefakty OCI (OCI = otwarty standard formatu obrazów) w jednym rejestrze |

**Poziomy usługi (SKU = wariant cenowy i funkcjonalny):** Basic, Standard, Premium. Większość codziennych operacji działa tak samo na każdym poziomie. Różnice (egzamin je lubi):

| Funkcja | Basic | Standard | Premium |
|---|---|---|---|
| Wliczony magazyn | 10 GiB | 100 GiB | 500 GiB |
| Geo-replikacja | nie | nie | tak |
| Prywatne punkty końcowe (private endpoints) | nie | nie | tak |
| Zaufanie do treści (content trust = podpisywanie obrazów) | nie | nie | tak |
| Retention policy (automatyczne usuwanie untagged manifests, czyli manifestów bez tagu; patrz C6) | nie | nie | tak |

Basic jest do nauki i rozwoju, Standard do większości produkcji, Premium do wdrożeń korporacyjnych. [docs: skus] Zmiana poziomu nie powoduje przestoju, ale przy przejściu z Premium na niższy trzeba najpierw usunąć np. geo-replikacje. [docs: skus]

Cele modułu (z unit 1): wyjaśnić hierarchię rejestr - repozytorium - artefakt; budować i zarządzać obrazami w chmurze przez ACR Tasks; wdrożyć strategie tagowania i wersjonowania; używać Azure CLI do zarządzania obrazami i uruchamiania szybkich zadań (quick tasks). [unit 1]

## A2. Hierarchia: rejestr, repozytorium, artefakt
_Źródło: [unit 2]_

ACR ma trzy poziomy:

```
rejestr  (contosoinference.azurecr.io)
  +-- repozytorium  (inference-api)
  |      +-- artefakt  inference-api:v1.2.0   (tagi, warstwy, manifest)
  |      +-- artefakt  inference-api:stable
  +-- repozytorium  (production/inference-api)
```

- **Rejestr (registry)** - zasób najwyższego poziomu. Ma unikalny **login server** w formacie `<nazwa-rejestru>.azurecr.io` (nazwa rejestru `contosoinference` daje `contosoinference.azurecr.io`). Tu ustawiasz uwierzytelnianie, dostęp (kto może push i pull, czyli wypychać i pobierać), retencję i geo-replikację.
- **Repozytorium (repository)** - kolekcja obrazów o tej samej nazwie i różnych tagach. Repozytorium `inference-api` zawiera wszystkie wersje obrazu API. Nazwa może mieć **przestrzenie nazw** (namespace) z ukośnikiem: `production/inference-api`, `staging/inference-api`, `ml-team/model-server`. Dzięki temu nadasz zespołowi prawo zapisu tylko do jego przestrzeni i zablokujesz obrazy produkcyjne.
- **Artefakt (artifact)** - właściwa zawartość w repozytorium: obraz Dockera, wykres Helm lub inna zgodna z OCI zawartość. Ma tagi, warstwy i manifest (patrz A3).

## A3. Tagi, warstwy i manifesty
_Źródło: [unit 2]_

- **Tag** - czytelna etykieta wersji w repozytorium, format `repozytorium:tag`, np. `inference-api:v1.2.0`. Bez podania tagu Docker używa `latest`. Jeden obraz może mieć wiele tagów (`v1.2.0` i `stable` wskazują ten sam obraz). **Tagi są mutowalne**: gdy wypchniesz nowy obraz z istniejącym tagiem, tag przesuwa się na nowy obraz. Przydatne dla `latest`, groźne na produkcji.
- **Warstwa (layer)** - obraz składa się z warstw; każda odpowiada instrukcji w Dockerfile (Dockerfile = plik z przepisem na obraz), która zmienia system plików (instalacja pakietu, kopiowanie plików). Warstwy są adresowane treścią (content-addressable): identyfikuje je skrót (hash) zawartości. ACR współdzieli wspólne warstwy między obrazami (**deduplikacja**): jeśli wiele obrazów ma bazę `python:3.11`, w rejestrze leży jedna kopia. To oszczędza miejsce i przyspiesza pobieranie.
- **Manifest** - dokument wymieniający warstwy i konfigurację artefaktu. Identyfikuje go **digest** w formacie `sha256:abc123...` (skrót SHA-256). **Digest jest niemutowalny**: po wypchnięciu obrazu nigdy się nie zmienia. Pobranie po digeście gwarantuje dokładnie ten obraz, nawet gdy ktoś wypchnie nowy obraz pod tym samym tagiem (egzamin je lubi).

## A4. Adresowanie obrazu: po tagu i po digeście
_Źródło: [unit 2]_

**Po tagu** - `rejestr/repozytorium:tag`. Czytelne, wygodne w konfiguracji wdrożeń, ale tag może się zmienić, więc wdrożenie nie jest w pełni powtarzalne.

```bash
docker pull myregistry.azurecr.io/inference-api:v1.2.0   # (1) pobranie po tagu
docker push myregistry.azurecr.io/inference-api:v1.2.0   # (2) wypchnięcie z tagiem
```

**Po digeście** - `rejestr/repozytorium@sha256:skrót`. Wskazuje konkretny, niezmienny obraz bez względu na tagi.

```bash
docker pull myregistry.azurecr.io/inference-api@sha256:0a2e01852872580b2c2fea9380ff8d7b637d3928783c55beb3f21a6e58d5d108   # (3)
```

Objaśnienie: (1) tag może wskazywać inny obraz jutro; (2) `push` wymaga wcześniejszego zalogowania do rejestru; (3) digest znajdziesz w portalu, w Azure CLI (`az acr manifest list-metadata`) albo w wyniku `push`. Digest stosuj tam, gdzie liczy się powtarzalność (produkcja, wszystkie węzły mają ten sam obraz).

## A5. Dobre praktyki organizacji rejestru
_Źródło: [unit 2]_

- **Używaj namespace** (zespół, środowisko, projekt) - upraszcza kontrolę dostępu i wyszukiwanie.
- **Zaplanuj strukturę repozytoriów** - grupuj obrazy według wzorca wdrożeń, właściciela lub cyklu życia.
- **Włącz geo-replikację** dla globalnych wdrożeń AI (Premium) - krótsze pobieranie, większa niezawodność.
- **Monitoruj magazyn** i stosuj polityki retencji, które usuwają untagged manifests (manifesty bez tagów). Bez sprzątania obrazy się kumulują i rosną koszty.

---

# CZĘŚĆ B - ACR TASKS: BUDOWANIE I URUCHAMIANIE OBRAZÓW W CHMURZE

## B1. Czym jest ACR Tasks
_Źródło: [unit 3] + [docs: https://learn.microsoft.com/en-us/azure/container-registry/container-registry-tasks-overview]_

**ACR Tasks** to zestaw funkcji, które przenoszą budowanie obrazów do Azure. Zamiast budować lokalnie i wypychać, wysyłasz kod źródłowy do ACR, a chmura wykonuje build. Korzyści:

- **Spójne buildy** - koniec z "u mnie działa": każdy build w tym samym środowisku.
- **Bez lokalnego Dockera** - nie musisz instalować Dockera na stacji roboczej.
- **Integracja CI/CD** (CI/CD = automatyczne budowanie, testowanie i wdrażanie) - wyzwalanie z commitów (GitHub, Azure DevOps) i aktualizacji obrazu bazowego.
- **Wiele platform** - Linux i Windows, także ARM (np. ARM64 dla urządzeń brzegowych).

Domyślnie ACR Tasks buduje dla Linux/AMD64; inną platformę wskazujesz parametrem `--platform` (np. `Linux/arm64/v8`). [docs: tasks-overview]

Trzy scenariusze: **quick tasks** (build na żądanie), **zadania wyzwalane automatycznie** (triggery) i **zadania wieloetapowe** (multi-step). Uwaga: uruchomienia ACR Tasks są tymczasowo wstrzymane dla darmowych kredytów Azure, więc ćwiczenie wymaga płatnej subskrypcji. [unit 5] [docs: tasks-overview]

## B2. Quick tasks: build na żądanie
_Źródło: [unit 3]_

Quick task buduje i wypycha pojedynczy obraz poleceniem `az acr build`. To odpowiednik `docker build` + `docker push`, ale w Azure.

```bash
az acr build --registry myregistry --image inference-api:v1.0.0 .   # (1)
```

Objaśnienie: (1) kropka na końcu to **kontekst builda** (build context = katalog ze źródłami; tu bieżący). ACR wysyła kontekst, buduje obraz według Dockerfile i wypycha go do rejestru.

**Skąd może pochodzić kontekst:**

- lokalny katalog (ACR kompresuje go i wysyła),
- repozytorium Git (publiczne lub prywatne, GitHub lub Azure DevOps) - podajesz adres URL,
- zdalny tarball (spakowane archiwum dostępne pod URL).

```bash
az acr build --registry myregistry \
  --image inference-api:v1.0.0 \
  https://github.com/myorg/inference-api.git    # (1) build bez klonowania repozytorium lokalnie
```

**Kiedy quick task:** weryfikacja zmian w Dockerfile przed commitem, build bez Dockera lokalnie, jednorazowy krok builda w CI/CD, testy nowych obrazów bazowych lub zależności.

## B3. Zadania wyzwalane automatycznie (triggery)
_Źródło: [unit 3] + [docs: https://learn.microsoft.com/en-us/azure/container-registry/container-registry-tasks-overview]_

**Trigger** (wyzwalacz) uruchamia build automatycznie, gdy zmieni się warunek. Trzy rodzaje (egzamin je lubi):

| Trigger | Co go uruchamia | Typowe użycie |
|---|---|---|
| Source code (kod źródłowy) | commit lub pull request w GitHub / Azure DevOps (ACR tworzy w repozytorium webhook = wywołanie HTTP po zdarzeniu) | CI po każdym commicie |
| Base image (obraz bazowy) | aktualizacja obrazu z instrukcji `FROM` | automatyczne łatki bezpieczeństwa |
| Scheduled (czasowy) | harmonogram w formacie cron (zapis czasu `minuta godzina dzień miesiąc dzień-tygodnia`) | nocne buildy, cykliczne sprzątanie |

**Source code trigger.** Zadanie tworzysz przez `az acr task create`. Znacznik `{{.Run.ID}}` daje unikalny tag dla każdego uruchomienia. Do dostępu do repozytorium ACR potrzebuje **osobistego tokena dostępu (PAT = personal access token)**, który przechowuj w Azure Key Vault, nie w skryptach.

```bash
az acr task create \
  --registry myregistry \
  --name build-inference-api \
  --image inference-api:{{.Run.ID}} \
  --context https://github.com/myorg/inference-api.git#main \
  --file Dockerfile \
  --git-access-token $PAT      # (1)
```

Objaśnienie: (1) token pozwala ACR czytać repozytorium i założyć webhook.

**Base image trigger.** Gdy zaktualizujesz obraz bazowy (np. `python:3.11` albo własny obraz z PyTorch), ACR wykrywa zmianę i przebudowuje wszystkie obrazy, które mają go w `FROM`. Działa dla obrazów bazowych w tym samym rejestrze albo w Docker Hub i innych rejestrach publicznych. Dla prywatnych obrazów bazowych baza i obraz aplikacji powinny być w tym samym rejestrze. [unit 3]

**Scheduled trigger** - przykład (codziennie o północy UTC):

```bash
az acr task create \
  --registry myregistry \
  --name nightly-build \
  --image inference-api:nightly \
  --context https://github.com/myorg/inference-api.git \
  --schedule "0 0 * * *" \
  --file Dockerfile \
  --git-access-token $PAT     # (1)
```

Objaśnienie: (1) pięć pól cron: minuta, godzina, dzień miesiąca, miesiąc, dzień tygodnia; `0 0 * * *` = 00:00 każdego dnia.

Zastosowania triggerów czasowych: nocne buildy z najnowszymi zależnościami, okresowe przebudowy (gdy wykrywanie zmian bazy nie zadziała), regularne skany bezpieczeństwa, zadania sprzątające stare untagged obrazy.

## B4. Zadania wieloetapowe (multi-step)
_Źródło: [unit 3] + [docs: https://learn.microsoft.com/en-us/azure/container-registry/container-registry-tasks-reference-yaml]_

**Multi-step task** rozszerza quick task o sekwencję kroków zapisaną w pliku YAML. Krok może zbudować obraz (`build`), wypchnąć go (`push`) albo uruchomić kontener lub polecenie (`cmd`). Dzięki temu cały cykl build - test - push mieści się w ACR.

```yaml
version: v1.1.0
steps:
  - build: -t {{.Run.Registry}}/inference-api:{{.Run.ID}} .          # (1)
  - push:
    - {{.Run.Registry}}/inference-api:{{.Run.ID}}                    # (2)
  - cmd: {{.Run.Registry}}/inference-api:{{.Run.ID}} python -m pytest tests/   # (3)
```

Objaśnienie: (1) budowa obrazu; (2) wypchnięcie do rejestru; (3) uruchomienie obrazu i testów (`pytest` = framework testów w Pythonie). Jeśli testy zawiodą, wiesz o tym zanim obraz trafi na wdrożenie. W dokumentacji używa się też aliasów `$Registry` i `$ID` zamiast `{{.Run.Registry}}` i `{{.Run.ID}}`. [docs: tasks-reference-yaml]

Multi-step obsługuje: workflowy build + test, sekwencję operacji z zależnościami między krokami, równoległe wykonanie niezależnych kroków i logikę warunkową zależną od wyniku poprzedniego kroku.

```bash
az acr task create \
  --registry myregistry \
  --name build-test-push \
  --context https://github.com/myorg/inference-api.git \
  --file acr-task.yaml \
  --git-access-token $PAT     # (1)
```

Objaśnienie: (1) zadanie z pliku YAML; samo `az acr run -f plik.yaml` uruchamia taki plik jednorazowo (patrz I3).

## B5. Uruchamianie obrazu: `az acr run`
_Źródło: [unit 3] + [docs: https://learn.microsoft.com/en-us/azure/container-registry/container-registry-tasks-reference-yaml]_

`az acr run` wykonuje polecenie w kontenerze z obrazu z Twojego rejestru. Pomaga wykryć błędy konfiguracji przed wdrożeniem. Kontekst `/dev/null` znaczy "żadne pliki źródłowe nie są potrzebne", bo uruchamiasz gotowy obraz.

```bash
az acr run --registry myregistry \
  --cmd 'myregistry.azurecr.io/inference-api:v1.0.0 python --version' \
  /dev/null
```

> **UWAGA - błąd w materiale Microsoftu** (unit 3): przykład `--cmd 'inference-api:v1.0.0 python --version'` podaje obraz bez nazwy rejestru -> krok `cmd` używa standardowego formatu `docker run`, a obraz bez rejestru jest traktowany jako pochodzący z docker.io; poprawnie podaj pełny adres `myregistry.azurecr.io/inference-api:v1.0.0` (tak robi też laboratorium w unit 5) albo `$Registry/inference-api:v1.0.0` w pliku YAML. Źródło: https://learn.microsoft.com/en-us/azure/container-registry/container-registry-tasks-reference-yaml.

Zastosowania: sprawdzenie, czy obraz startuje; czy są potrzebne narzędzia i frameworki; testy dymne (smoke tests = szybkie sprawdzenia podstawowej sprawności); polecenia kontroli stanu.

## B6. Dobre praktyki ACR Tasks
_Źródło: [unit 3]_

- **Zmienne uruchomienia** - `{{.Run.ID}}` lub `{{.Run.Date}}` w tagach dają unikalne, śledzone buildy.
- **Zabezpiecz tokeny** - PAT w Azure Key Vault, nie w repozytorium ani skryptach.
- **Logi zadań** - `az acr task logs` pokazuje cały przebieg builda z błędami.
- **Kontekst builda** - plik `.dockerignore` wyklucza zbędne pliki; duży kontekst spowalnia build i zużywa transfer.
- **Cache warstw** - ACR Tasks zapamiętuje warstwy między buildami; ustaw instrukcje Dockerfile tak, by często zmieniane były na końcu pliku.

---

# CZĘŚĆ C - TAGOWANIE, WERSJONOWANIE, BLOKADY I SPRZĄTANIE

## C1. Stabilne i unikalne tagi
_Źródło: [unit 4]_

**Stabilne tagi** (`v1`, `v1.2`, `latest`) są używane wielokrotnie: nowy obraz z istniejącym tagiem przejmuje tag, a poprzedni obraz zostaje w rejestrze bez tego tagu. Pasują do: obrazów bazowych z łatkami bezpieczeństwa, środowisk deweloperskich (zawsze najnowsze zmiany) i ciągłego dostarczania (continuous delivery = każda zmiana automatycznie trafia do odbiorców). Wada: brak przewidywalności. Różne węzły mogą pobrać ten sam tag w różnym czasie i dostać różne obrazy, a dla usługi wnioskowania AI to oznacza różne wersje modelu.

**Unikalne tagi** (`v1.2.0-build456`, `20260102-abc123`) nigdy nie są ponownie używane; każdy push tworzy nowy tag i zachowuje wszystkie wersje. Pasują do: wdrożeń produkcyjnych (każdy węzeł ma ten sam obraz), ścieżki audytu, cofania do znanego dobrego stanu (rollback) i wymogów zgodności. Wada: przy każdym wydaniu trzeba zaktualizować konfigurację wdrożenia, ale na produkcji to zaleta, bo zmiana jest świadoma.

## C2. Wersjonowanie semantyczne
_Źródło: [unit 4]_

**Semantic versioning** to format `MAJOR.MINOR.PATCH`:

- **MAJOR** - zmiany łamiące zgodność (zmiana API, usunięta funkcja),
- **MINOR** - nowe funkcje zgodne wstecz,
- **PATCH** - poprawki błędów i bezpieczeństwa bez zmiany API.

```text
inference-api:1.0.0    # pierwsze wydanie
inference-api:1.0.1    # poprawka błędu w przetwarzaniu wstępnym
inference-api:1.1.0    # nowy endpoint modelu
inference-api:2.0.0    # zmiana łamiąca API
```

Połącz wersje semantyczne ze stabilnymi tagami, żeby odbiorcy wybrali strategię aktualizacji:

```text
inference-api:1        # wskazuje najnowszą 1.x.x  (tag stabilny)
inference-api:1.1      # wskazuje najnowszą 1.1.x  (tag stabilny)
inference-api:1.1.0    # konkretna wersja          (tag unikalny)
```

Kto używa `inference-api:1`, dostaje aktualizacje w obrębie wersji głównej; kto `inference-api:1.1.0`, dostaje tylko tę wersję.

## C3. Wzorce unikalnych tagów
_Źródło: [unit 4]_

| Wzorzec | Przykład | Po co |
|---|---|---|
| ID builda z systemu CI/CD | `inference-api:build-4567` | Łączy obraz z uruchomieniem pipeline (pipeline = zautomatyzowany ciąg kroków builda i testów; logi, wyniki testów) |
| Skrót commitu Git (commit SHA) | `inference-api:abc123f` | Najprostsze powiązanie obrazu z kodem źródłowym; każdy z dostępem do repozytorium sprawdzi kod |
| Znacznik czasu | `inference-api:20260102-143022` | Widać wiek i kolejność obrazów |
| Połączony | `inference-api:v1.2.0-build4567-abc123f` | Pełna śledzalność przy incydencie |

W ACR Tasks zmienna `{{.Run.ID}}` generuje unikalny identyfikator każdego builda:

```bash
az acr build --registry myregistry \
  --image inference-api:v1.2.0-{{.Run.ID}} .     # (1)
```

Objaśnienie: (1) tag będzie np. `v1.2.0-ca3` (ID uruchomienia zadania ACR).

## C4. Tag `latest`
_Źródło: [unit 4]_

Gdy nie podasz tagu, Docker użyje `latest`. Na produkcji to pułapka (egzamin je lubi):

- **niespójne wdrożenia** - węzły pobierają `latest` w różnych momentach i dostają różne obrazy,
- **nieprzewidywalne aktualizacje** - wdrożenie zmienia się, gdy ktoś wypchnie nowy obraz, bez świadomego wdrożenia,
- **trudna diagnostyka** - nie wiesz, jaka wersja działa.

```yaml
# Unikaj na produkcji
image: myregistry.azurecr.io/inference-api:latest

# Używaj jawnej wersji
image: myregistry.azurecr.io/inference-api:v1.2.0
```

Jeśli workflow wymaga `latest`, używaj go tylko w środowiskach deweloperskich.

## C5. Blokowanie wdrożonych obrazów
_Źródło: [unit 4] + [docs: https://learn.microsoft.com/en-us/azure/container-registry/container-registry-image-lock]_

Domyślnie otagowany obraz w ACR jest **mutowalny** (możesz go nadpisać i usunąć). Dla obrazu działającego na produkcji chcesz **niemutowalności**. Blokadę ustawiasz przez atrybuty poleceniem `az acr repository update`:

```bash
az acr repository update \
  --name myregistry \
  --image inference-api:v1.2.0 \
  --write-enabled false      # (1) blokada

az acr repository update \
  --name myregistry \
  --image inference-api:v1.2.0 \
  --write-enabled true       # (2) odblokowanie przed wycofaniem wersji
```

Zablokowany obraz: **nie da się go usunąć** (nawet administrator przypadkiem), **nie da się go nadpisać** (push z tym samym tagiem kończy się błędem), **przeżywa polityki retencji** i daje pewność, że obraz produkcyjny pozostanie dostępny. [unit 4]

Z dokumentacji [docs: image-lock]: atrybuty `write-enabled`, `delete-enabled` i `list-enabled` ustawiasz dla wersji obrazu (tag lub digest) albo dla całego repozytorium (`--repository`). `delete-enabled=false` chroni przed usunięciem, ale pozwala na aktualizację; `list-enabled=false` blokuje listowanie. Atrybuty tagu i jego manifestu są zarządzane osobno: ustawienie `deleteEnabled=false` na tagu nie ustawia go na manifeście. Blokada rejestru jako zasobu (Azure Locks, `az lock`) to co innego: nie chroni danych w repozytoriach.

## C6. Sprzątanie: untagged images, `acr purge`, retention policy
_Źródło: [unit 4] + [docs: https://learn.microsoft.com/en-us/azure/container-registry/container-registry-auto-purge] + [docs: https://learn.microsoft.com/en-us/azure/container-registry/container-registry-retention-policy]_

Gdy wypchniesz nowy obraz pod istniejącym stabilnym tagiem, poprzedni obraz traci tag i staje się **untagged** (manifest bez tagu). Takie "osierocone" obrazy zajmują miejsce i rosną koszty.

**`acr purge`** to polecenie działające jako kontener w ACR Tasks. Uruchamiasz je na żądanie przez `az acr run`:

```bash
az acr run --registry myregistry \
  --cmd "acr purge --filter 'inference-api:.*' --untagged --ago 30d" \
  /dev/null
```

Zaplanowane sprzątanie (zadanie co tydzień, niedziela 00:00):

```bash
az acr task create \
  --registry myregistry \
  --name cleanup-untagged \
  --cmd "acr purge --filter '.*:.*' --untagged --ago 7d" \
  --schedule "0 0 * * 0" \
  --context /dev/null
```

> **UWAGA - błąd w materiale Microsoftu** (unit 4): moduł opisuje oba polecenia jako usuwanie tylko untagged images starszych niż 30 dni (7 dni), a drugie jako dotyczące "untagged images across all repositories" -> `acr purge --filter 'repo:tag' --ago N` **usuwa także tagi** pasujące do filtra i starsze niż N; `--untagged` dodatkowo usuwa manifesty bez tagów. Filtr `'.*:.*'` z `--ago 7d` dopasowuje wszystkie tagi we wszystkich repozytoriach, więc zaplanowane zadanie co tydzień kasuje wszystkie obrazy starsze niż 7 dni (poza zablokowanymi). Do usuwania wyłącznie untagged użyj `--untagged-only` albo filtra bez dopasowania tagów (`^$`). Źródło: https://learn.microsoft.com/en-us/azure/container-registry/container-registry-auto-purge.

Co jeszcze mówi dokumentacja [docs: auto-purge] (egzamin je lubi):

- `--filter` to **wyrażenie regularne nazwy repozytorium i wyrażenie regularne nazwy tagu** w postaci `repo:tag`, a nie sama nazwa repozytorium (`'hello-world:.*'` = wszystkie tagi repozytorium `hello-world`),
- `--ago` to czas w stylu Go: `d` dni, `h` godziny, `m` minuty (np. `2d3h6m`),
- `--dry-run` pokazuje, co zostałoby usunięte, bez usuwania - używaj przed każdym realnym purge,
- `--keep N` zachowuje N ostatnich pozycji,
- domyślnie `acr purge` **pomija zablokowane** tagi i manifesty (`write-enabled` lub `delete-enabled` = false); `--include-locked` odblokowuje je i usuwa (dostępne od `mcr.microsoft.com/acr/acr-cli:0.17`),
- usunięte dane są **nieodwracalne**.

**Retention policy** (polityka retencji) to prostsza alternatywa dla zaplanowanego purge: jedno ustawienie na cały rejestr usuwa untagged manifests po zadanej liczbie dni, bez zadań i harmonogramów. Dla rejestrów **Premium**. Dokumentacja dodaje: to funkcja w wersji **preview** (podglądowa); domyślny okres to 7 dni (zakres 0-365; 0 = natychmiast po odtagowaniu); dotyczy tylko manifestów odtagowanych po włączeniu polityki; nie obsługuje manifestów w typach OCI (tylko manifesty Dockera) - do nich użyj `acr purge`; nie ustawiaj jej, jeśli systemy pobierają obrazy po digeście, bo untagged obraz zniknie. Manifest z `delete-enabled=false` jest z polityki wyłączony. [docs: retention-policy] Stan na 2026-11-22; sprawdź status preview w aktualnej dokumentacji przed użyciem.

## C7. Dobre praktyki tagowania i wersjonowania
_Źródło: [unit 4]_

- **Unikalne tagi na produkcji** - spójność na wszystkich węzłach.
- **Stabilne tagi tylko dla obrazów bazowych** - łatki płyną do zależnych obrazów przez base image triggers.
- **Blokuj obrazy produkcyjne** - odblokuj dopiero przy wycofywaniu wersji.
- **Polityki retencji i regularny purge** - kontrola kosztów magazynu.
- **Metadane builda w tagu** - ID builda i commit ułatwiają diagnostykę i audyt.
- **Udokumentuj schemat tagowania** - zespół ma wiedzieć, które tagi są stabilne, a które unikalne.

## C8. Zarządzanie obrazami z kodu: SDK dla Pythona i .NET
_Źródło: [docs: https://learn.microsoft.com/en-us/python/api/overview/azure/containerregistry-readme] + [docs: https://learn.microsoft.com/en-us/dotnet/api/overview/azure/containers.containerregistry-readme] + [docs: https://learn.microsoft.com/en-us/azure/container-registry/container-registry-roles] + [autor: test 2026-11-22]_

Nazwy klas i metod poniżej sprawdzono 2026-11-22 introspekcją zainstalowanego pakietu (Python) i kompilacją projektu .NET; wywołań na prawdziwym rejestrze nie testowano.

Egzamin AI-200 sprawdza biegłość w **Azure SDK** i w Pythonie, więc oprócz `az` warto znać bibliotekę klienta rejestru (**SDK** = Software Development Kit, biblioteka do wywoływania usługi z kodu). Moduł Microsoftu uczy tego przez CLI; ta sekcja uzupełnia go o odpowiedniki w kodzie.

| Zadanie | Python (`azure-containerregistry`) | C# (`Azure.Containers.ContainerRegistry`) |
|---|---|---|
| Klient | `ContainerRegistryClient(endpoint, DefaultAzureCredential(), audience=...)` | `new ContainerRegistryClient(uri, new DefaultAzureCredential(), options)` |
| Lista repozytoriów | `list_repository_names()` | `GetRepositoryNamesAsync()` |
| Lista manifestów | `list_manifest_properties(repo)` | `GetRepository(repo).GetAllManifestPropertiesAsync()` |
| Blokada obrazu | `update_manifest_properties(repo, tag_or_digest, can_write=False)` | `GetArtifact(repo, tag).UpdateManifestPropertiesAsync(new ArtifactManifestProperties { CanWrite = false })` |

Ważne (egzamin je lubi): uwierzytelnianie przez Microsoft Entra ID (`DefaultAzureCredential` bierze np. dane z `az login`); w chmurze publicznej ustaw **audience** na `https://management.azure.com` (Python) albo `ContainerRegistryAudience.AzureResourceManagerPublicCloud` (.NET). `can_write=False` w SDK odpowiada `--write-enabled false` w CLI. Do operacji na obrazach potrzebna jest rola danych w rejestrze (np. `Container Registry Repository Writer` do wypychania i zmian, `Container Registry Repository Contributor` do usuwania), nadawana osobno od ról zarządzania rejestrem. [docs: roles]

> Wersje sprawdzone 2026-11-22 (źródło: https://pypi.org/project/azure-containerregistry/, https://www.nuget.org/packages/Azure.Containers.ContainerRegistry): Python `azure-containerregistry` 1.2.0, `azure-identity` 1.27.0; .NET `Azure.Containers.ContainerRegistry` 1.4.0. Moduł Microsoftu nie podaje wersji SDK; na egzaminie liczy się zasada (klient + `DefaultAzureCredential` + właściwy audience). Przed użyciem sprawdź PyPI i NuGet.

---

# CZĘŚĆ D - ĆWICZENIA

Przed ćwiczeniami przeczytaj części A-C. Każde ćwiczenie ma układ: Gdzie / Plik, kod lub kroki, Koszt, Zadania, Sprawdź, Sprzątanie. I1 jest laboratorium z modułu (unit 5) w wersji niezależnej od pobranego archiwum; I2-I4 są autorskie i korzystają z rejestru utworzonego w I1 (zmienne `RG` i `ACR_NAME`). Wszystkie ćwiczenia tworzą płatny rejestr ACR; wykonaj Sprzątanie z ostatniego ćwiczenia, które robisz.

> Niesprawdzone: dokładna cena rejestru ACR Basic za dzień - zweryfikuj w https://azure.microsoft.com/pricing/details/container-registry/. Poziom Basic wlicza 10 GiB magazynu [docs: skus]; builds ACR Tasks są dodatkowo naliczane za czas obliczeń.

## I1. Build, weryfikacja, uruchomienie i blokada obrazu przez ACR Tasks (laboratorium modułu)
_Źródło: [unit 5] + [unit 5-lab] + [unit 5-starter] + [autor: niezweryfikowane]_
(rozwinięte w `01-store-manage-containers-azure-container-registry.md` §B2, §B5, §A3, §C5)

**Gdzie / Plik:** Cloud Shell (Bash), katalog roboczy `acr-lab`. Pliki aplikacji: `acr-lab/api/app.py` i `acr-lab/api/Dockerfile` (to pliki startowe laboratorium Microsoftu, podane tu w całości, żeby nie pobierać archiwum). Wymaga subskrypcji płatnej (nie darmowych kredytów).

Krok 1 - aplikacja (Python, Flask = lekki framework WWW; endpoint = adres URL, pod którym API odpowiada; tu `/`, `/health`, `/predict`):

```python
# acr-lab/api/app.py
from flask import Flask, jsonify
import os

app = Flask(__name__)


@app.route('/health')
def health():
    return jsonify({"status": "healthy", "version": os.getenv("APP_VERSION", "1.0.0")})        # (1)


@app.route('/predict')
def predict():
    return jsonify({"prediction": "sample-result", "confidence": 0.95,
                    "model_version": os.getenv("MODEL_VERSION", "v1")})                        # (2)


@app.route('/')
def root():
    return jsonify({"name": "Inference API", "version": os.getenv("APP_VERSION", "1.0.0"),
                    "endpoints": ["/health", "/predict"]})


if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)                                                         # (3)
```

Objaśnienie: (1) `/health` to endpoint kontroli stanu dla orkiestratorów (np. Kubernetes); wersja z zmiennej środowiskowej `APP_VERSION`; (2) udaje endpoint wnioskowania - w produkcji wywołałby model; (3) `host='0.0.0.0'` pozwala na połączenia spoza kontenera, port 5000.

```dockerfile
# acr-lab/api/Dockerfile
FROM python:3.11-slim
WORKDIR /app
RUN pip install --no-cache-dir flask
COPY app.py .
ENV APP_VERSION=1.0.0
ENV MODEL_VERSION=v1
EXPOSE 5000
CMD ["python", "app.py"]
```

Krok 2 - środowisko i rejestr (SKU Basic, jak w skrypcie laboratorium):

```bash
mkdir -p acr-lab/api && cd acr-lab                                  # utwórz pliki app.py i Dockerfile w api/ (powyżej)
az login
az provider register --namespace Microsoft.ContainerRegistry        # (1)
export RG=rg-acr-i1
export ACR_NAME=acrtut$RANDOM$RANDOM                                # (2)
az group create --name $RG --location westeurope
az acr create --resource-group $RG --name $ACR_NAME --sku Basic     # (3)
```

Objaśnienie: (1) rejestruje dostawcę zasobów, jeśli subskrypcja go jeszcze nie ma; (2) nazwa rejestru musi być globalnie unikalna (litery i cyfry, 5-50 znaków); (3) rejestr w poziomie Basic.

Krok 3 - build w chmurze i weryfikacja:

```bash
az acr build --registry $ACR_NAME --image inference-api:v1.0.0 ./api          # (1)
az acr repository list --name $ACR_NAME --output table                        # (2)
az acr repository show-tags --name $ACR_NAME --repository inference-api --output table   # (3)
az acr manifest list-metadata --registry $ACR_NAME --name inference-api --output table   # (4)
```

Objaśnienie: (1) wysyła kontekst `./api`, buduje i wypycha obraz bez Dockera lokalnie; wynik pokazuje etapy: spakowanie i wysłanie kontekstu, kolejkowanie, strumień budowy warstw, wypchnięcie, digest i status; (2) lista repozytoriów; (3) lista tagów; (4) metadane manifestów razem z digestem.

Krok 4 - uruchomienie obrazu i kolejna wersja:

```bash
az acr run --registry $ACR_NAME \
  --cmd "$ACR_NAME.azurecr.io/inference-api:v1.0.0 python -c 'from app import app'" \
  /dev/null                                                                    # (1)
az acr build --registry $ACR_NAME --image inference-api:v1.1.0 ./api           # (2)
az acr repository show-tags --name $ACR_NAME --repository inference-api --output table
```

Objaśnienie: (1) sprawdza, że aplikacja Flask importuje się w kontenerze (pełny adres obrazu, patrz uwaga o błędzie w B5); (2) druga wersja obrazu, ta sama zawartość.

Krok 5 - historia i blokada:

```bash
az acr task list-runs --registry $ACR_NAME --output table                      # (1)
az acr repository update --name $ACR_NAME --image inference-api:v1.0.0 --write-enabled false   # (2)
az acr repository show --name $ACR_NAME --image inference-api:v1.0.0           # (3)
```

Objaśnienie: (1) historia uruchomień (ID, status, trigger, czas); (2) blokada obrazu `v1.0.0`; (3) w wyniku pole `writeEnabled` ma wartość `false`.

Gdy coś nie działa (rozwiązywanie problemów z laboratorium Microsoftu): sprawdź `az account show` (czy jesteś w dobrej subskrypcji) i `echo $ACR_NAME` (czy zmienna jest ustawiona; po otwarciu nowego terminala ustaw ją ponownie); w portalu sprawdź, czy rejestr ma stan wdrożenia **Succeeded**, albo uruchom `az acr list --output table`; przy błędach builda sprawdź, czy jesteś w katalogu zawierającym folder `api`, czy istnieje Dockerfile i poprawne ścieżki, a stan buildów obejrzyj przez `az acr task list-runs --registry $ACR_NAME --output table`. [unit 5-lab]

**Koszt:** rejestr ACR Basic jest płatny i naliczany za dzień istnienia (cena: patrz ostrzeżenie "Niesprawdzone" na początku części D); builds ACR Tasks są naliczane za czas obliczeń. Przy kilkudziesięciu minutach pracy koszt jest niewielki, o ile wykonasz Sprzątanie. Darmowe kredyty Azure nie obsługują uruchomień ACR Tasks. [unit 5]

**Zadania:** (a) zbuduj `v1.0.0` i odczytaj z wyniku digest; (b) wypisz repozytoria i tagi; (c) uruchom `az acr run` i potwierdź sukces; (d) zbuduj `v1.1.0` i sprawdź, że oba tagi istnieją; (e) zablokuj `v1.0.0` i odczytaj `writeEnabled`; (f) **zepsuj to:** spróbuj zbudować jeszcze raz `inference-api:v1.0.0` (to samo polecenie co w kroku 3).

**Sprawdź:** (a) w wyniku jest linia z digestem `sha256:...`; (b) widoczne repozytorium `inference-api` i tag `v1.0.0`; (c) wynik kończy się komunikatem w stylu `Run ID: xxx was successful after xxx`; (d) lista tagów zawiera `v1.0.0` i `v1.1.0`; (e) `writeEnabled` = `false`; (f) build kończy się błędem przy wypychaniu, bo zablokowanego tagu nie można nadpisać (dokładny komunikat zależy od wersji ACR - [autor: niezweryfikowane]).

**Sprzątanie:**
```bash
az group delete --name rg-acr-i1 --yes --no-wait
az group exists --name rg-acr-i1     # po kilku minutach powinno wypisać false
```
Uwaga: usunięcie grupy zasobów usuwa wszystkie zasoby w niej (rejestr i obrazy). Rejestr z I1 jest używany w I2-I4, więc wykonaj sprzątanie po ostatnim ćwiczeniu, które robisz.

## I2. Unikalne tagi, tagi stabilne i pobranie po digeście - Python i C#
_Źródło: [unit 4] + [docs: https://learn.microsoft.com/en-us/azure/container-registry/container-registry-image-tag-version] + [autor: niezweryfikowane]_

**Gdzie / Plik:** Cloud Shell (Bash), katalog `acr-lab` z I1 (aplikacja Pythona w `api/`); rejestr z I1 (`RG`, `ACR_NAME`). Dla bloku C# nowy katalog `acr-lab/api-dotnet` z trzema plikami (poniżej).

Krok 1 - unikalny tag z numerem uruchomienia oraz tagi stabilne (jeden build, wiele tagów `--image`):

```bash
export RG=rg-acr-i1
export ACR_NAME=$(az acr list --resource-group $RG --query "[0].name" -o tsv)       # (1)

az acr build --registry $ACR_NAME \
  --image inference-api:v1.2.0-{{.Run.ID}} \
  --image inference-api:1.2 \
  --image inference-api:1 \
  ./api                                                                              # (2)

az acr repository show-tags --name $ACR_NAME --repository inference-api --output table
DIGEST=$(az acr manifest list-metadata --registry $ACR_NAME --name inference-api \
  --query "[?tags && contains(tags, '1.2')].digest | [0]" -o tsv)                     # (3)
echo $DIGEST
az acr run --registry $ACR_NAME \
  --cmd "$ACR_NAME.azurecr.io/inference-api@$DIGEST python -c 'from app import app; print(\"ok\")'" \
  /dev/null                                                                           # (4)
```

Objaśnienie: (1) odczyt nazwy rejestru z grupy, jeśli nie masz już zmiennej; (2) jeden obraz dostaje trzy tagi: unikalny `v1.2.0-<RunID>` (np. `v1.2.0-ca3`) oraz stabilne `1.2` i `1` (patrz C2); (3) wyszukanie digestu obrazu z tagiem `1.2` (zapytanie JMESPath = język zapytań `--query` w `az`); (4) uruchomienie obrazu po **digeście** - gwarantuje dokładnie ten obraz niezależnie od tagów.

**Kod Pythona do tego ćwiczenia.** Krok 1 buduje aplikację z `./api`, czyli kod z I1, tylko z nowymi tagami. Dla wygody ten sam kod jest tu jeszcze raz. Jeśli zrobiłeś I1, pliki już masz i nic nie zmieniasz (tag `v1.2.0-...` to ten sam kod; zmieniają się tylko tagi).

```python
# acr-lab/api/app.py
from flask import Flask, jsonify
import os

app = Flask(__name__)


@app.route('/health')
def health():
    return jsonify({"status": "healthy", "version": os.getenv("APP_VERSION", "1.0.0")})

@app.route('/predict')
def predict():
    return jsonify({"prediction": "sample-result", "confidence": 0.95,
                    "model_version": os.getenv("MODEL_VERSION", "v1")})

@app.route('/')
def root():
    return jsonify({"name": "Inference API", "version": os.getenv("APP_VERSION", "1.0.0"),
                    "endpoints": ["/health", "/predict"]})


if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

```dockerfile
# acr-lab/api/Dockerfile
FROM python:3.11-slim
WORKDIR /app
RUN pip install --no-cache-dir flask
COPY app.py .
ENV APP_VERSION=1.0.0
ENV MODEL_VERSION=v1
EXPOSE 5000
CMD ["python", "app.py"]
```

Poniżej odpowiednik tej samej aplikacji w C#.

**Odpowiednik C# (.NET).** Aplikacja ASP.NET Core (minimal API = bez kontrolerów, endpointy w kilku liniach) z tymi samymi trzema endpointami. Dockerfile jest wieloetapowy (osobno SDK do budowy, osobno lekki runtime), ale sposób budowania w ACR Tasks jest taki sam.

Plik `acr-lab/api-dotnet/InferenceApi.csproj`:

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
// acr-lab/api-dotnet/Program.cs
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

string appVersion = Environment.GetEnvironmentVariable("APP_VERSION") ?? "1.0.0";    // (1)
string modelVersion = Environment.GetEnvironmentVariable("MODEL_VERSION") ?? "v1";  // (2)

app.MapGet("/", () => new { name = "Inference API", version = appVersion, endpoints = new[] { "/health", "/predict" } });
app.MapGet("/health", () => new { status = "healthy", version = appVersion });
app.MapGet("/predict", () => new { prediction = "sample-result", confidence = 0.95, model_version = modelVersion });

app.Run();                                                                            // (3)
```

```dockerfile
# acr-lab/api-dotnet/Dockerfile
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src
COPY InferenceApi.csproj .
RUN dotnet restore
COPY . .
RUN dotnet publish -c Release -o /out --no-restore

FROM mcr.microsoft.com/dotnet/aspnet:10.0
WORKDIR /app
COPY --from=build /out .
ENV ASPNETCORE_URLS=http://+:5000
ENV APP_VERSION=1.0.0
ENV MODEL_VERSION=v1
EXPOSE 5000
CMD ["dotnet", "InferenceApi.dll"]
```

Objaśnienie: (1)-(2) zmienne środowiskowe czytane tak samo jak `os.getenv` w Pythonie; (3) uruchomienie serwera (w Pythonie robi to `app.run`); `ASPNETCORE_URLS` ustawia port 5000, tak jak w wersji Pythona.

```bash
az acr build --registry $ACR_NAME --image inference-api-dotnet:v1.0.0 ./api-dotnet
az acr run --registry $ACR_NAME \
  --cmd "$ACR_NAME.azurecr.io/inference-api-dotnet:v1.0.0 dotnet --list-runtimes" \
  /dev/null
```

Różnice względem Pythona: build wieloetapowy zamiast jednego `pip install`; zamiast "import aplikacji" smoke test sprawdza dostępność runtime .NET (`dotnet --list-runtimes`); tagi, digesty i polecenia `az acr` są identyczne. Wersje obrazów bazowych `mcr.microsoft.com/dotnet/sdk:10.0` i `aspnet:10.0` istnieją w rejestrze MCR (sprawdzone 2026-11-22), podobnie `python:3.12-slim` w Docker Hub; I1 używa `python:3.11-slim` jak laboratorium.

**Koszt:** dodatkowe buildy ACR Tasks na istniejącym rejestrze (czas obliczeń, groszowe kwoty); brak nowych zasobów.

**Zadania:** (a) zbuduj obraz z trzema tagami i wypisz tagi; (b) odczytaj digest i uruchom obraz po digeście; (c) zbuduj aplikację C# w tym samym rejestrze i uruchom smoke test; (d) **zepsuj to:** w `az acr run` podaj obraz bez nazwy rejestru (`inference-api:1.2`) i przeczytaj błąd; (e) wypisz repozytoria rejestru.

**Sprawdź:** (a) widać tag `v1.2.0-<RunID>` oraz `1.2` i `1`; (b) komenda wypisuje `ok`; (c) lista runtime'ów zawiera `Microsoft.AspNetCore.App 10.0.x` i `Microsoft.NETCore.App 10.0.x`; (d) bez rejestru obraz jest szukany w docker.io i uruchomienie kończy się błędem pobrania obrazu (patrz uwaga o błędzie w B5; dokładny komunikat - [autor: niezweryfikowane]); (e) są dwa repozytoria: `inference-api` i `inference-api-dotnet`.

**Sprzątanie:**
```bash
az acr repository delete --name $ACR_NAME --image inference-api-dotnet:v1.0.0 --yes   # opcjonalnie: sam obraz C#
az group delete --name rg-acr-i1 --yes --no-wait                                       # pełne sprzątanie po ostatnim ćwiczeniu
az group exists --name rg-acr-i1     # po kilku minutach: false
```

## I3. Zadanie wieloetapowe i bezpieczny `acr purge` z `--dry-run`
_Źródło: [unit 3] + [unit 4] + [docs: https://learn.microsoft.com/en-us/azure/container-registry/container-registry-auto-purge] + [autor: niezweryfikowane]_

**Gdzie / Plik:** Cloud Shell (Bash), katalog `acr-lab`; plik `acr-lab/acr-task.yaml`; rejestr z I1 i obrazy z I1-I2.

```yaml
# acr-lab/acr-task.yaml
version: v1.1.0
steps:
  - build: -t $Registry/inference-api:v1.3.0-$ID ./api         # (1)
  - push:
    - $Registry/inference-api:v1.3.0-$ID                       # (2)
  - cmd: $Registry/inference-api:v1.3.0-$ID python -c "from app import app; print('test ok')"   # (3)
```

```bash
az acr run --registry $ACR_NAME -f acr-task.yaml .             # (4)

# dry-run: pokaż, co usunąłby purge, bez usuwania
az acr run --registry $ACR_NAME \
  --cmd "acr purge --filter 'inference-api:.*' --ago 0d --untagged --dry-run" \
  /dev/null                                                    # (5)

# bezpieczny wariant: tylko untagged
az acr run --registry $ACR_NAME \
  --cmd "acr purge --filter 'inference-api:^\$' --ago 0d --untagged --dry-run" \
  /dev/null                                                    # (6)
```

Objaśnienie: (1) krok `build` z aliasem `$Registry` (login server) i `$ID` (ID uruchomienia); kontekst `./api` to katalog z Dockerfile, bo cały plik uruchamiasz z kropką jako kontekstem (4); (2) wypchnięcie; (3) krok `cmd` uruchamia właśnie zbudowany obraz i sprawdza import aplikacji; (4) jednorazowe uruchomienie pliku YAML bez tworzenia zadania; (5) `--dry-run` wypisuje tagi i manifesty, które zostałyby usunięte: widzisz, że filtr `inference-api:.*` z `--ago 0d` obejmuje **wszystkie tagi** (patrz uwaga o błędzie w C6); zablokowany `v1.0.0` jest pomijany; (6) filtr tagu `^$` nie pasuje do żadnego tagu, więc dry-run pokazuje tylko untagged manifests.

**Koszt:** buildy ACR Tasks na istniejącym rejestrze (czas obliczeń); `--dry-run` nic nie usuwa.

**Zadania:** (a) uruchom plik YAML i potwierdź, że kroki build, push i cmd zakończyły się sukcesem; (b) wykonaj dry-run z filtrem `inference-api:.*` i zapisz, które tagi byłyby usunięte; (c) wykonaj dry-run z filtrem `inference-api:^\$` i porównaj listę; (d) **zepsuj to:** zmień w pliku YAML test w kroku `cmd` na `from app import nieistnieje` i uruchom ponownie.

**Sprawdź:** (a) log pokazuje trzy kroki i `test ok`; (b) lista zawiera tagi starsze niż 0 dni, z wyjątkiem zablokowanego `v1.0.0`; (c) lista zawiera tylko manifesty bez tagów (może być pusta); (d) krok `cmd` kończy się błędem (`ImportError`), a całe zadanie ma status Failed - tak wiesz o problemie zanim obraz trafi na wdrożenie. [autor: niezweryfikowane]

**Sprzątanie:** brak dodatkowych zasobów (dry-run nic nie usuwa). Jeśli to ostatnie ćwiczenie, usuń grupę: `az group delete --name rg-acr-i1 --yes --no-wait`, potem `az group exists --name rg-acr-i1` (po kilku minutach: `false`).

## I4. Rejestr z kodu: lista obrazów i blokada przez SDK - Python i C#
_Źródło: [docs: https://learn.microsoft.com/en-us/python/api/overview/azure/containerregistry-readme] + [docs: https://learn.microsoft.com/en-us/dotnet/api/overview/azure/containers.containerregistry-readme] + [docs: https://learn.microsoft.com/en-us/azure/container-registry/container-registry-roles] + [autor: niezweryfikowane]_

**Gdzie / Plik:** terminal lokalny z Azure CLI, Python 3.10+ i .NET SDK 10 (lub Cloud Shell z Pythonem). Pliki: `acr-lab/sdk/inspect_acr.py` (Python) oraz `acr-lab/sdk/AcrInspect/` (C#). Rejestr z I1 i obraz `inference-api:v1.1.0`. Kod skompilowano/sprawdzono składniowo 2026-11-22 [autor: niezweryfikowane] - nie był uruchamiany na prawdziwym rejestrze.

Krok 0 - uprawnienia i środowisko:

```bash
az login
export RG=rg-acr-i1
export ACR_NAME=$(az acr list --resource-group $RG --query "[0].name" -o tsv)
ME=$(az ad signed-in-user show --query id -o tsv)
ACR_ID=$(az acr show --name $ACR_NAME --query id -o tsv)
az role assignment create --assignee $ME --role "Container Registry Repository Writer" --scope $ACR_ID   # (1)
python -m venv .venv && source .venv/bin/activate         # Windows: .venv\Scripts\activate
pip install azure-containerregistry==1.2.0 azure-identity      # (2)
```

Objaśnienie: (1) rola danych potrzebna do zmiany atrybutów obrazów przez Entra ID; propagacja roli może potrwać kilka minut [docs: roles]; jeśli masz rolę Owner na subskrypcji, zarządzanie rejestrem już masz, ale uprawnienia do danych nadaje się osobno; (2) wersja 1.2.0 sprawdzona 2026-11-22.

Krok 1 - kod Pythona:

```python
# acr-lab/sdk/inspect_acr.py
import os

from azure.containerregistry import ContainerRegistryClient
from azure.identity import DefaultAzureCredential

endpoint = f"https://{os.environ['ACR_NAME']}.azurecr.io"        # (1)
audience = "https://management.azure.com"                         # (2)

with ContainerRegistryClient(endpoint, DefaultAzureCredential(), audience=audience) as client:  # (3)
    for repo in client.list_repository_names():                   # (4)
        print("repozytorium:", repo)
        for m in client.list_manifest_properties(repo):           # (5)
            print("  digest:", m.digest)
            print("  tagi:", m.tags, " can_write:", m.can_write)

    props = client.update_manifest_properties(                    # (6)
        "inference-api", "v1.1.0", can_write=False
    )
    print("po blokadzie: can_write =", props.can_write)
```

Objaśnienie: (1) adres rejestru z nazwy w zmiennej środowiskowej; (2) audience dla chmury publicznej (tokeny Entra dla zarządzania Azure); (3) `with` zamyka klienta; `DefaultAzureCredential` użyje m.in. sesji `az login`; (4) iterator nazw repozytoriów; (5) właściwości manifestów: digest, tagi, uprawnienia; (6) blokada zapisu (odpowiednik `--write-enabled false`).

```bash
python acr-lab/sdk/inspect_acr.py
```

**Odpowiednik C# (.NET).** Ten sam przepływ w pakiecie `Azure.Containers.ContainerRegistry` 1.4.0.

Plik `acr-lab/sdk/AcrInspect/AcrInspect.csproj`:

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net10.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="Azure.Containers.ContainerRegistry" Version="1.4.0" />
    <PackageReference Include="Azure.Identity" Version="1.21.0" />
  </ItemGroup>
</Project>
```

```csharp
// acr-lab/sdk/AcrInspect/Program.cs
using Azure.Containers.ContainerRegistry;
using Azure.Identity;

var acrName = Environment.GetEnvironmentVariable("ACR_NAME")
    ?? throw new InvalidOperationException("Ustaw zmienną ACR_NAME.");        // (1)
var endpoint = new Uri($"https://{acrName}.azurecr.io");                      // (2)
var client = new ContainerRegistryClient(
    endpoint,
    new DefaultAzureCredential(),                                             // (3)
    new ContainerRegistryClientOptions
    {
        Audience = ContainerRegistryAudience.AzureResourceManagerPublicCloud  // (4)
    });

await foreach (string repoName in client.GetRepositoryNamesAsync())           // (5)
{
    Console.WriteLine($"repozytorium: {repoName}");
    ContainerRepository repo = client.GetRepository(repoName);
    await foreach (ArtifactManifestProperties m in repo.GetAllManifestPropertiesAsync())
    {
        Console.WriteLine($"  digest: {m.Digest}");
        Console.WriteLine($"  tagi: {string.Join(", ", m.Tags)}  canWrite: {m.CanWrite}");
    }
}

RegistryArtifact image = client.GetArtifact("inference-api", "v1.1.0");      // (6)
await image.UpdateManifestPropertiesAsync(new ArtifactManifestProperties { CanWrite = false }); // (7)
ArtifactManifestProperties after = await image.GetManifestPropertiesAsync();
Console.WriteLine($"po blokadzie: canWrite = {after.CanWrite}");
```

Objaśnienie: (1)-(2) endpoint rejestru; (3) `DefaultAzureCredential` jak w Pythonie; (4) odpowiednik `audience`; (5) `await foreach` iteruje stronicowane wyniki asynchronicznie (w Pythonie wersja synchroniczna); (6) obiekt `RegistryArtifact` wskazuje obraz po tagu lub digeście; (7) blokada zapisu. Różnice względem Pythona: API asynchroniczne i typowane obiekty (`RegistryArtifact`, `ContainerRepository`), a nie jedna klasa klienta z metodami `*_properties`.

```bash
cd acr-lab/sdk/AcrInspect && dotnet run
```

**Koszt:** brak nowych zasobów; wywołania SDK są darmowe, ale rejestr z I1 nadal jest płatny do czasu Sprzątania.

**Zadania:** (a) uruchom skrypt Pythona i zapisz listę repozytoriów, digestów i tagów; (b) potwierdź w CLI, że obraz `v1.1.0` jest zablokowany; (c) uruchom wersję C# i porównaj wynik; (d) **zepsuj to:** uruchom skrypt bez `audience` (usuń argument `audience=...`) albo usuń rolę z kroku 0 i przeczytaj błąd.

**Sprawdź:** (a) widoczne repozytoria `inference-api` (i `inference-api-dotnet`, jeśli zrobiłeś I2), a przy każdym manifeście digest i tagi; (b) `az acr repository show --name $ACR_NAME --image inference-api:v1.1.0` pokazuje `writeEnabled: false`; (c) wynik zgodny z Pythonem, `canWrite = False`; (d) bez właściwej roli lub z niezgodnym audience uwierzytelnienie lub operacja kończy się błędem autoryzacji (kod 401/403; dokładny komunikat - [autor: niezweryfikowane]).

**Sprzątanie:**
```bash
deactivate 2>/dev/null; rm -rf .venv acr-lab/sdk/AcrInspect/bin acr-lab/sdk/AcrInspect/obj
az role assignment delete --assignee $ME --role "Container Registry Repository Writer" --scope $ACR_ID
az group delete --name rg-acr-i1 --yes --no-wait
az group exists --name rg-acr-i1     # po kilku minutach: false
```

---

# CZĘŚĆ E - QUIZ Z MODUŁU MICROSOFTU (5 pytań) + wyjaśnienia

Learn pokazuje pytania, ale nie pokazuje poprawnych odpowiedzi, więc każda odpowiedź poniżej jest **wnioskowana** z treści jednostek.

## E1. Spójne buildy (zespół buduje obrazy na laptopach)
-> *ACR Tasks quick build.* Buduje obrazy w kontrolowanym środowisku Azure (B1, B2). Pułapki: geo-replikacja rozprowadza obrazy do regionów, nie ujednolica buildów; przestrzenie nazw repozytoriów organizują obrazy i dostęp.
*(odpowiedź wnioskowana - Learn nie pokazuje odpowiedzi; uzasadnienie: unit 3, sekcje "Understand ACR Tasks" i "Quick tasks")*

## E2. Ten sam obraz na każdym węźle, nawet po wypchnięciu nowego z tym samym tagiem
-> *Po manifest digest.* Digest jest niemutowalny (A3, A4). Pułapki: `latest` i tag semantyczny (jeśli nie jest unikalny) mogą wskazać inny obraz po ponownym wypchnięciu.
*(odpowiedź wnioskowana - Learn nie pokazuje odpowiedzi; uzasadnienie: unit 2, sekcje "Manifests" i "By digest")*

## E3. Automatyczny rebuild po łatce w obrazie bazowym PyTorch
-> *Base image update trigger.* Wykrywa zmianę obrazu z `FROM` i przebudowuje zależne obrazy (B3). Pułapki: trigger kodu reaguje na commity, czasowy na harmonogram, a nie na aktualizację bazy.
*(odpowiedź wnioskowana - Learn nie pokazuje odpowiedzi; uzasadnienie: unit 3, sekcja "Base image triggers")*

## E4. Tagowanie produkcyjne ze śledzeniem do commitu i rollbackiem
-> *Unikalne tagi z commit SHA Git.* Dają połączenie z kodem źródłowym i możliwość cofnięcia do dowolnej wersji (C1, C3). Pułapki: stabilne tagi `v1`, `v2` się przesuwają, a samo `latest` nie daje ani śledzenia, ani rollbacku.
*(odpowiedź wnioskowana - Learn nie pokazuje odpowiedzi; uzasadnienie: unit 4, sekcje "Unique tags" i "Git commit SHA tags")*

## E5. Ochrona obrazu produkcyjnego przed przypadkowym usunięciem
-> *Blokada obrazu przez `write-enabled false`.* (C5). Pułapki: przestrzenie nazw i geo-replikacja nie chronią przed usunięciem.
*(odpowiedź wnioskowana - Learn nie pokazuje odpowiedzi; uzasadnienie: unit 4, sekcja "Lock deployed images")*

---

# CZĘŚĆ F - PYTANIA KONTROLNE (odpowiedz bez zaglądania)

## F1. Wymień trzy poziomy hierarchii w ACR i podaj format login servera.
<details><summary>Odpowiedź</summary>

Rejestr (login server `<nazwa>.azurecr.io`), repozytorium (kolekcja obrazów o tej samej nazwie z różnymi tagami, z możliwością przestrzeni nazw przez `/`) i artefakt (obraz, wykres Helm lub inna zawartość OCI z tagami, warstwami i manifestem).
</details>

## F2. Czym różni się tag od digestu i kiedy używać którego?
<details><summary>Odpowiedź</summary>

Tag jest czytelną, mutowalną etykietą (można go przesunąć na nowy obraz); digest (`sha256:...`) jest niemutowalnym skrótem manifestu. Tag wygodny w rozwoju, digest (lub unikalny tag) tam, gdzie liczy się powtarzalność, np. produkcja.
</details>

## F3. Jakie są trzy rodzaje triggerów ACR Tasks i co uruchamia każdy?
<details><summary>Odpowiedź</summary>

Source code (commit lub pull request w GitHub/Azure DevOps), base image (aktualizacja obrazu z `FROM`) i scheduled (harmonogram cron).
</details>

## F4. Do czego służy `az acr build`, a do czego `az acr run`?
<details><summary>Odpowiedź</summary>

`az acr build` to quick task: wysyła kontekst, buduje obraz w chmurze i wypycha do rejestru (bez lokalnego Dockera). `az acr run` uruchamia polecenie w kontenerze z obrazu (np. test dymny) lub plik YAML zadania; obraz w `--cmd` podawaj z pełnym adresem rejestru.
</details>

## F5. Czym różnią się tagi stabilne i unikalne i które stosować na produkcji?
<details><summary>Odpowiedź</summary>

Stabilne (`v1`, `latest`) są ponownie używane i przesuwają się na nowy obraz; unikalne (np. `v1.2.0-build456`, commit SHA) nigdy nie są używane drugi raz. Na produkcji unikalne, bo dają spójność, audyt i rollback; stabilne dla obrazów bazowych i środowisk dev.
</details>

## F6. Jak zablokować obraz w ACR i co to blokuje?
<details><summary>Odpowiedź</summary>

`az acr repository update --name <rejestr> --image <repo:tag> --write-enabled false`. Obraz nie może być usunięty ani nadpisany, `acr purge` go pomija (bez `--include-locked`) i polityka retencji go nie usuwa. Odblokowanie: `--write-enabled true`.
</details>

## F7. Co robi `acr purge --filter 'repo:.*' --untagged --ago 30d`? (pułapka)
<details><summary>Odpowiedź</summary>

Usuwa tagi pasujące do filtra (wszystkie tagi repozytorium `repo`) starsze niż 30 dni oraz dodatkowo manifesty bez tagów. Nie usuwa zablokowanych (bez `--include-locked`). Do samych untagged użyj `--untagged-only`. Przed realnym uruchomieniem użyj `--dry-run`; usunięte dane są nieodwracalne.
</details>

## F8. Na jakim poziomie ACR działa retention policy dla untagged manifests i jakie ma ograniczenia?
<details><summary>Odpowiedź</summary>

Tylko na poziomie Premium. Według dokumentacji jest w wersji preview, domyślnie 7 dni (zakres 0-365), dotyczy manifestów odtagowanych po włączeniu polityki i tylko manifestów Dockera (nie OCI). Nie używaj jej, gdy systemy pobierają obrazy po digeście.
</details>

## F9. Które poziomy ACR obsługują geo-replikację i private endpoints?
<details><summary>Odpowiedź</summary>

Tylko Premium (geo-replikacja, prywatne punkty końcowe, content trust). Basic i Standard ich nie mają; różnią się wliczonym magazynem (10 GiB i 100 GiB).
</details>

---

# CZĘŚĆ G - CO DALEJ

## G1. Podsumowanie, następny krok i pokrycie egzaminu
_Źródło: [unit 7] + [docs: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/ai-200]_

**Podsumowanie modułu.** ACR organizuje obrazy w hierarchii rejestr - repozytorium - artefakt; manifesty, warstwy i digesty dają wydajne przechowywanie i precyzyjną identyfikację. Tagi to czytelne referencje, digesty to niemutowalne identyfikatory. ACR Tasks buduje obrazy w chmurze bez lokalnego Dockera: quick tasks na żądanie, triggery (commit, obraz bazowy, harmonogram) i zadania wieloetapowe build - test - push. Wersjonowanie semantyczne mówi o charakterze zmian, unikalne tagi dają spójność wdrożeń, a blokada chroni obrazy produkcyjne. [unit 7]

**Co dalej:** moduł 02 - `deploy-containers-azure-app-service` (https://learn.microsoft.com/en-us/training/modules/deploy-containers-azure-app-service/): wdrażanie obrazów z ACR do Azure App Service (porty, polecenia startowe, ustawienia aplikacji i sekrety, diagnostyka). Wróć do A4 i C4 przy wyborze tagu w konfiguracji App Service oraz do B5, gdy chcesz sprawdzić obraz przed wdrożeniem.

**Pokrycie egzaminu** - lista umiejętności AI-200 (stan study guide z 2026-10-30), domena 1, część "Implement container application hosting":

| Umiejętność (study guide) | Domena | Status | Gdzie |
|---|---|---|---|
| Build, store, version, and manage container images by using Azure Container Registry | 1 | uczone tutaj | `01-store-manage-containers-azure-container-registry.md` §A1-A5, §C1-C8, §I1-I2 |
| Build and run images by using Azure Container Registry Tasks | 1 | uczone tutaj | `01-store-manage-containers-azure-container-registry.md` §B1-B6, §I1, §I3 |
| Deploy containers to Azure App Service, including configuring App Service to supply environment variables and secrets | 1 | uczone w module 02 | https://learn.microsoft.com/en-us/training/modules/deploy-containers-azure-app-service/ (plik 02 jeszcze nie istnieje) |

Pozostałe umiejętności egzaminu (m.in. Container Apps, AKS i domeny 2-4) należą do innych modułów kursu; pełny przydział jest w `sources/_exam/skill-map.md`. Żaden punkt study guide nie jest orphanem, więc plik luk egzaminu nie jest potrzebny.

## G2. Uczciwie o egzaminie
_Źródło: [autor]_

Ten moduł pokrywa dwie z trzech umiejętności tzw. "Implement container application hosting": zarządzanie obrazami w ACR i ACR Tasks. Trzecia (App Service ze zmiennymi środowiskowymi i sekretami) jest w module 02. Na egzaminie spodziewaj się pytań o: wybór triggera, tagi stabilne czy unikalne, pobieranie po digeście, blokadę obrazu, poziomy ACR (co jest tylko w Premium) oraz o składnię `az acr build`/`run`/`task create`. Moduł uczy głównie przez Azure CLI; ponieważ egzamin dotyczy Azure SDK i Pythona, przećwicz też I4. Błędy w materiale Microsoftu opisane w tym pliku (B5, C6) sprawdź w dokumentacji, zanim zaufasz przykładom z modułu. Nie przerobiłeś jeszcze: App Service (moduł 02), sidecarów (moduł 03) oraz modułów Container Apps i AKS.
