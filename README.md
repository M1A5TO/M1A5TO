# M1A5TO

To repo spina moduły projektu (BACKEND, SCRAPPER, POI, AI, FUZZY oraz FRONTEND) i umożliwia uruchomienie całości lokalnie (Docker).

Projekt M1A5TO jest rozwijany jako zestaw repozytoriów w organizacji GitHub: https://github.com/M1A5TO

## Krótki opis / Short description

**PL:** Celem projektu jest zbudowanie aplikacji webowej (serwisu), która na podstawie danych podanych przez użytkownika wyszukuje mieszkania na terenie Polski i analizuje je w kontekście koncepcji 15‑minutowego miasta, dodatkowo określając ich kluczowe parametry (cena, metraż, dostępność do POI, wyniki dla określonych profili, analizy zdjęć w ofertach).

**EN:** The goal of the project is to build a web application (service) which, based on user-provided input, searches for apartments across Poland and analyzes them in the context of the 15-minute city concept, additionally determining their key parameters (price, size, accessibility to POI, profile-based score, image analysis).

## Repositories / Repozytoria

Linki do źródłowych repozytoriów:

- **FRONTEND:** https://github.com/M1A5TO/Web-Application
- **SCRAPPER (real-estate listing scraper):** https://github.com/M1A5TO/realestate-scraper
- **BACKEND:** https://github.com/M1A5TO/Backend
- **AI (interior image classifier):** https://github.com/M1A5TO/AI-interior-image-classifier
- **POI (15-minute city algorithm):** https://github.com/M1A5TO/15MC-Algorithm
- **FUZZY (fuzzy logic for user profiles):** https://github.com/M1A5TO/Fuzzy-Logic-Profiles

## Uruchomienie lokalne (Docker)

### Konfiguracja

1. Skopiuj `.env.example` → `.env` i (opcjonalnie) zmień wartości.
2. Upewnij się, że Docker Desktop działa.

### Podstawowe uruchomienie

**Cały projekt:**
```bash
docker compose up -d
```

**Tylko frontend (+ wymagane serwisy):**
```bash
docker compose up -d frontend
```

## Szczegóły modułów

### POI – przygotowanie danych dla całej Polski

> **Wymagane przed scrapowaniem na całą Polskę**

Jeżeli scrapper działa na "całą Polskę", to **worker POI musi mieć gotowe artefakty** (graf/POI/precompute) dla kafelków obejmujących obszar, w którym pojawią się mieszkania.

Worker POI wybiera kafelek na podstawie pliku `GRID_JSON` (w compose: `POI_GRID_JSON`, domyślnie `data/grid_test.json`), a następnie oczekuje artefaktów w:

- `POI/workspace/<grid_id>/`

**Minimalny zestaw plików na kafelek:**
- graf pieszy: `*_csr.npz`
- POI snapped do node'ów: `pois.parquet`
- precompute reachability: `*_precompute.npz` (opcjonalnie także summary CSV)

Jeżeli dla danego punktu (lat/lon) nie ma kafelka w `GRID_JSON` lub brakuje artefaktów w `POI/workspace/<grid_id>/`, worker zwróci błąd dla tej wiadomości.

**Szczegółowa instrukcja przygotowania danych** (grid → tile extraction → graph/pois/precompute) jest w repo `POI/README.md`.

### AI – analiza zdjęć (etap po POI)

W pipeline M1A5TO etap AI jest uruchamiany jako serwis `ai-worker` w `compose.yml`.

#### Co robi AI

1. Konsumuje wiadomości `{ "apartment_id": <id> }` z kolejki RabbitMQ `poi_results`
2. Pobiera z backendu listę zdjęć mieszkania (przez `GET /apartments/{id}` i `GET /photos/{photo_id}`)
3. Pobiera obrazy po URL (`photo.link`) i rozpoznaje:
   - `photo_type` (interior / non-interior)
   - `room_type`
   - `style` / `room_style`
4. Aktualizuje zdjęcia w backendzie `PUT /photos/{photo_id}`
5. Publikuje wynik na kolejkę `image_classification_results`

#### Wymagania

- `POI` musi publikować do `poi_results` (serwis `poi-worker`)
- Mieszkania muszą mieć zdjęcia w backendzie (`photo_ids` przy apartamencie + dostępne `/photos/{id}`), inaczej AI nic nie zaktualizuje
- Plik wag LoRA jest montowany z hosta: `AI/lora_models/comprehensive_lora.pth`

#### Konfiguracja

AI w tej integracji nie wymaga dodatkowych zmiennych w `.env` (host RabbitMQ i URL API są ustawione w `compose.yml`).

### FUZZY – scoring atrakcyjności (etap po AI)

W pipeline M1A5TO etap FUZZY jest uruchamiany jako serwis `fuzzy-worker` w `compose.yml`.

#### Co robi FUZZY

1. Konsumuje wiadomości `{ "apartment_id": <id> }` z kolejki RabbitMQ (domyślnie `image_classification_results`)
2. Pobiera z backendu mieszkanie (`GET /apartments/{id}`) oraz relacje POI (`GET /apartments/{id}/pois`)
3. Wylicza scoring profilowy (student/singiel/rodzina/właściciel psa/uniwersalny) i opisy (`poi_desc`, `price_desc`, `size_desc`)
4. Zapisuje wyniki do backendu przez `PUT /apartments/{id}`

#### Konfiguracja

FUZZY jest konfigurowany zmiennymi środowiskowymi (polecane: w `.env`):

- `FUZZY_INPUT_QUEUE` (domyślnie `image_classification_results`)
- `FUZZY_TIME_UNIT` (`seconds`|`minutes`, domyślnie `seconds`)
- `FUZZY_ALPHA` (0..1, domyślnie `0.9`, wyższe = większy wpływ POI)

Parametry scoringu są zdefiniowane w `FUZZY/score_apartments_offline.py`.

#### Wymagania

- `POI` musi być policzone i zapisane w backendzie (worker `poi-worker`)
- `AI` powinno wykonać klasyfikację zdjęć i opublikować `image_classification_results` (worker `ai-worker`)
