# M1A5TO (local integration)

To repo jest "superprojektem" spinającym submodule: BACKEND, SCRAPPER, POI, AI, FUZZY.

## Uruchomienie lokalne (Windows + Docker Desktop)

### 1) Konfiguracja
1. Skopiuj `.env.example` → `.env` i (opcjonalnie) zmień wartości.
2. Upewnij się, że Docker Desktop działa.

### 2) Start infrastruktury i API
Uruchom z katalogu głównego repo:
- `docker compose up -d postgres rabbitmq api`

Po starcie:
- Swagger: http://localhost:8081/docs
- RabbitMQ UI: http://localhost:15672 (login/hasło z `.env`)

### 2.1) Start FRONTEND (UI)
Frontend jest uruchamiany jako serwis `frontend` (Vite dev server):
- `docker compose up -d frontend`

Po starcie:
- UI: http://localhost:5173

Frontend w trybie lokalnym używa proxy Vite:
- requesty do `/api/*` są przekierowywane do backendu w sieci dockera (`http://api:8000`)

### 3) Start scrapperów (tryb live)
Możesz uruchomić wszystkie 4 scrappery jednocześnie:
- `docker compose up -d scrapper-otodom scrapper-morizon scrapper-gratka scrapper-trojmiasto`

Każdy scrapper wykona `live` z limitem stron `SCRAPPER_MAX_PAGES` (domyślnie 200) i zakończy pracę.

Alternatywnie uruchamiaj je jednorazowo (polecane do testów):
- `docker compose run --rm scrapper morizon live --city "Gdańsk" --limit 5`

## POI – przygotowanie danych dla całej Polski (wymagane przed scrapowaniem)

Jeżeli scrapper działa na "całą Polskę", to **worker POI musi mieć gotowe artefakty** (graf/POI/precompute) dla kafelków obejmujących obszar, w którym pojawią się mieszkania.

Worker POI wybiera kafelek na podstawie pliku `GRID_JSON` (w compose: `POI_GRID_JSON`, domyślnie `data/grid_test.json`), a następnie oczekuje artefaktów w:

- `POI/workspace/<grid_id>/`

Minimalny zestaw plików na kafelek:
- graf pieszy: `*_csr.npz`
- POI snapped do node'ów: `pois.parquet`
- precompute reachability: `*_precompute.npz` (opcjonalnie także summary CSV)

Jeżeli dla danego punktu (lat/lon) nie ma kafelka w `GRID_JSON` lub brakuje artefaktów w `POI/workspace/<grid_id>/`, worker zwróci błąd dla tej wiadomości.

**Szczegółowa instrukcja przygotowania danych** (grid → tile extraction → graph/pois/precompute) jest w repo `POI/README.md`.

## AI – analiza zdjęć (etap po POI)

W pipeline M1A5TO etap AI jest uruchamiany jako serwis `ai-worker` w `compose.yml`.

### Co robi AI
1. konsumuje wiadomości `{ "apartment_id": <id> }` z kolejki RabbitMQ `poi_results`
2. pobiera z backendu listę zdjęć mieszkania (przez `GET /apartments/{id}` i `GET /photos/{photo_id}`)
3. pobiera obrazy po URL (`photo.link`) i rozpoznaje:
   - `photo_type` (interior / non-interior)
   - `room_type`
   - `style` / `room_style`
4. aktualizuje zdjęcia w backendzie `PUT /photos/{photo_id}`
5. publikuje wynik na kolejkę `image_classification_results`

### Wymagania
- `POI` musi publikować do `poi_results` (serwis `poi-worker`).
- mieszkania muszą mieć zdjęcia w backendzie (`photo_ids` przy apartamencie + dostępne `/photos/{id}`), inaczej AI nic nie zaktualizuje.
- plik wag LoRA jest montowany z hosta: `AI/lora_models/comprehensive_lora.pth`.

### Konfiguracja
AI w tej integracji nie wymaga dodatkowych zmiennych w `.env` (host RabbitMQ i URL API są ustawione w `compose.yml`).

## FUZZY – scoring atrakcyjności (etap po AI)

W pipeline M1A5TO etap FUZZY jest uruchamiany jako serwis `fuzzy-worker` w `compose.yml`.

### Co robi FUZZY
1. konsumuje wiadomości `{ "apartment_id": <id> }` z kolejki RabbitMQ (domyślnie `image_classification_results`)
2. pobiera z backendu mieszkanie (`GET /apartments/{id}`) oraz relacje POI (`GET /apartments/{id}/pois`)
3. wylicza scoring profilowy (student/singiel/rodzina/właściciel psa/uniwersalny) i opisy (`poi_desc`, `price_desc`, `size_desc`)
4. zapisuje wyniki do backendu przez `PUT /apartments/{id}`

### Konfiguracja
FUZZY jest konfigurowany zmiennymi środowiskowymi (polecane: w `.env`):
- `FUZZY_INPUT_QUEUE` (domyślnie `image_classification_results`)
- `FUZZY_TIME_UNIT` (`seconds`|`minutes`, domyślnie `seconds`)
- `FUZZY_ALPHA` (0..1, domyślnie `0.9`, wyższe = większy wpływ POI)

Parametry scoringu są zdefiniowane w `FUZZY/score_apartments_offline.py`.

### Wymagania
- `POI` musi być policzone i zapisane w backendzie (worker `poi-worker`).
- `AI` powinno wykonać klasyfikację zdjęć i opublikować `image_classification_results` (worker `ai-worker`).

### Uruchomienie
Dla pełnego pipeline uruchamiaj w kolejności:
1) `docker compose up -d postgres rabbitmq api`
2) `docker compose up -d poi-worker`
3) `docker compose up -d ai-worker`
4) `docker compose up -d fuzzy-worker`
5) `docker compose up -d scrapper-otodom scrapper-morizon scrapper-gratka scrapper-trojmiasto`

> Uwaga: scrapper jest źródłem danych, ale uruchamiamy go na końcu celowo.
> Dzięki temu workery (POI/AI/FUZZY) już nasłuchują kolejek i przetwarzanie startuje od razu,
> bez kumulowania backlogu w RabbitMQ.
