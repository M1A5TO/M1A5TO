# M1A5TO – raport/sprawozdanie końcowe

## 1. Cel projektu

M1A5TO jest modularnym systemem (aplikacja webowa wraz z pipeline’em przetwarzania danych), który pełni rolę serwisu przeznaczonego do wyszukiwania oraz oceny mieszkań w Polsce w kontekście koncepcji tzw. **15‑minutowego miasta**.

Głównym celem projektu jest umożliwienie użytkownikowi analizy atrakcyjności ofert mieszkaniowych na podstawie zarówno cech samego lokalu (cena, metraż, zdjęcia), jak i jego otoczenia. Użytkownik podaje lokalizację oraz wybiera profil preferencji (np. student, rodzina), natomiast system:

- pozyskuje (batchowo) oferty mieszkań i ich dane z internetowych serwisów ogłoszeniowych,
- analizuje dostępność kluczowych punktów (POI) w zasięgu dojścia pieszego w 15 minut,
- przetwarza zdjęcia wnętrz w celu identyfikacji typu pomieszczeń oraz cech stylistycznych,
- oblicza końcowy wskaźnik atrakcyjności mieszkania przy użyciu logiki rozmytej,
- prezentuje wyniki w aplikacji webowej.

System jest rozwijany w sposób modułowy. Przetwarzanie zostało przygotowane zarówno do pracy rozproszonej (synchronizacja zdalna), jak i do lokalnego uruchomienia całego stosu.

## 2. Zakres danych i założenia projektowe

### 2.1 Dane o mieszkaniach (SCRAPPER)

Źródłem danych są publicznie dostępne serwisy ogłoszeniowe z rynku nieruchomości (m.in. Otodom, Gratka, Morizon, Trójmiasto.pl).

Analizowane są wyłącznie mieszkania przeznaczone na sprzedaż. Minimalny zakres danych obejmuje **cenę**, **lokalizację**, **metraż** oraz **fotografie wnętrz**. Oferty niekompletne są automatycznie odrzucane na etapie walidacji.

Dodatkowo w module zaimplementowano mechanizm wykrywania i odrzucania duplikatów ofert.

System projektowany jest z myślą o skali ogólnopolskiej, jednak testy funkcjonalne i metryki jakościowe realizowane są na obszarze Gdańska.

### 2.2 Punkty POI i koncepcja 15‑minutowego miasta

Punkty zainteresowania (POI) obejmują m.in. sklepy, szkoły, placówki medyczne (szpitale, apteki, przychodnie), a także inne obiekty (np. parki, bary, siłownie) w zależności od profilu użytkownika.

Dojście piesze modelowane jest w sposób uproszczony poprzez przyjęcie stałej prędkości marszu **4 km/h** oraz maksymalnego dystansu rzędu **~1 km**, odpowiadającego około 15 minutom spaceru.

Dane POI pochodzą z **OpenStreetMap** i są pobierane jednorazowo. Ze względu na ograniczenia zasobowe system nie wykonuje obliczeń w czasie rzeczywistym – analiza mapy i przygotowanie artefaktów (graf/kafelki) odbywa się offline przed etapem integracji/synchronizacji.

### 2.3 Atrakcyjność i profile użytkowników

Końcowy wynik atrakcyjności mieszkania wyznaczany jest na podstawie agregacji kilku grup czynników: dostępności punktów POI, ceny, metrażu oraz wyników analizy zdjęć.

Dla różnych profili użytkowników (np. student, singiel, rodzina, właściciel psa, profil uniwersalny) stosowane są odmienne wagi poszczególnych kryteriów, co pozwala generować zróżnicowane rankingi.

## 3. Architektura systemu i przepływ danych

System składa się z kilku niezależnych modułów, zintegrowanych poprzez wspólny backend oraz mechanizm kolejek wiadomości.

Komponenty systemu:
- **SCRAPPER**
- **POI**
- **AI**
- **FUZZY**
- **BACKEND**
- **FRONTEND**

Typowy przepływ danych:
1. **SCRAPPER** pozyskuje oferty mieszkań i zapisuje je w backendzie.
2. **POI** pobiera dane mieszkań, oblicza metryki dostępności POI i aktualizuje bazę danych.
3. **AI** analizuje zdjęcia wnętrz (klasyfikacja + cechy/styl) i zapisuje wyniki w backendzie.
4. **FUZZY** agreguje dane i oblicza końcowy wskaźnik atrakcyjności dla profili użytkowników.
5. **FRONTEND** pobiera gotowe dane przez API i prezentuje je użytkownikowi.

Integracja pomiędzy etapami pipeline’u realizowana jest przy użyciu wzorca workerów oraz kolejek wiadomości **RabbitMQ**, które pełnią rolę mechanizmu wyzwalającego kolejne etapy przetwarzania.

## 4. Opis modułów i zastosowany stack technologiczny

### 4.1 SCRAPPER – pozyskiwanie ofert mieszkań

Moduł SCRAPPER odpowiada za automatyczne pobieranie ofert mieszkań z różnych serwisów internetowych oraz ich normalizację. Jego zadaniem jest ujednolicenie formatu danych, walidacja kompletności ofert oraz przygotowanie danych do dalszego przetwarzania w systemie.

Zakres funkcjonalny obejmuje obsługę wielu źródeł, pobieranie kluczowych informacji (m.in. cena, metraż, lokalizacja, zdjęcia), normalizację danych, odrzucanie ofert niekompletnych, wykrywanie duplikatów oraz buforowanie wyników w formie CSV (we wcześniejszych etapach), a docelowo przesyłanie danych bezpośrednio do backendu.

**Stack technologiczny:**
Moduł został zaimplementowany w języku Python z wykorzystaniem m.in. Scrapy, scrapy-playwright, BeautifulSoup, selectolax, pandas, pydantic oraz narzędzi do obsługi retry, logowania i konfiguracji środowiska.

### 4.2 POI – analiza przestrzenna i algorytm 15‑minutowy

Moduł POI odpowiada za analizę danych **OpenStreetMap** i wyznaczanie dostępności punktów usługowych w zasięgu dojścia pieszego. W tym celu budowany jest graf pieszy na podstawie danych OSM, a obszar przetwarzania dzielony jest na kafelki w celu optymalizacji obliczeń.

Przetwarzanie mapy odbywa się offline przed uruchomieniem trybu zdalnego lub lokalnego, a dane OSM nie są aktualizowane w trakcie projektu. Wyniki obliczeń zapisywane są w backendzie.

**Stack technologiczny:**
Moduł POI bazuje na danych OpenStreetMap (OSM) oraz narzędziach do ich ekstrakcji i filtrowania (np. osmium). Przetwarzanie wykonywane jest w Pythonie, gdzie dane mapowe konwertowane są do reprezentacji grafowej (np. z użyciem biblioteki networkx). Następnie wykorzystywany jest algorytm heurystyczny do wyznaczania tras pieszych i obliczania dostępności POI w zasięgu „15 minut”. Wyniki zapisywane są do bazy przez API backendu.

### 4.3 AI – analiza zdjęć wnętrz

Moduł AI realizuje automatyczną analizę zdjęć mieszkań. Jego zadaniem jest rozróżnianie zdjęć wnętrz od innych ujęć oraz klasyfikacja typu pomieszczenia i cech stylistycznych, a także określenie stylu całego mieszkania.

W module stosowane jest podejście oparte na transfer learningu, w szczególności z wykorzystaniem modelu **CLIP** z fine‑tuningiem.

**Stack technologiczny:**
Moduł analizy zdjęć został przygotowany w Pythonie. Wykorzystuje model CLIP (z fine‑tuningiem), a do przetwarzania danych używa bibliotek takich jak NumPy i pandas. Pobiera zdjęcia po URL, generuje metadane (np. typ pomieszczenia, cechy/styl) i zapisuje wyniki do backendu.

### 4.4 FUZZY – scoring atrakcyjności

Moduł FUZZY agreguje dane o mieszkaniu, wyniki analizy POI oraz cechy wyodrębnione ze zdjęć, a następnie oblicza końcowy wskaźnik atrakcyjności dla różnych profili użytkowników oraz może generować tagi opisujące mieszkanie.

Do obliczeń wykorzystywana jest logika rozmyta (Sugeno), a wyniki zapisywane są w backendzie.

**Stack technologiczny:**
Moduł scoringu profilowego został zaimplementowany w Pythonie. Do obliczeń numerycznych wykorzystywany jest NumPy, a do realizacji logiki rozmytej biblioteka scikit-fuzzy. Moduł pobiera dane mieszkania i metryki POI z backendu, liczy atrakcyjność dla profili (rodzina/student/singiel/właściciel psa/uniwersalny), generuje tagi i zapisuje wyniki do bazy.

### 4.5 BACKEND – API i baza danych

Backend pełni rolę centralnego punktu integracji całego systemu. Odpowiada za przechowywanie informacji o mieszkaniach, zdjęciach, wynikach analizy POI oraz atrakcyjności.

Moduł udostępnia REST API umożliwiające filtrowanie i pobieranie danych, realizuje geokodowanie adresów oraz obsługuje komunikację z workerami przetwarzającymi kolejne etapy pipeline’u.

**Stack technologiczny:**
Moduł backend został zaimplementowany w języku Python z wykorzystaniem frameworka FastAPI do budowy REST API. Warstwa danych opiera się o PostgreSQL z rozszerzeniem PostGIS do przechowywania i zapytań geolokalizacyjnych. Całość uruchamiana jest w kontenerach Docker (Docker Compose) i pełni rolę centralnego punktu integracji dla scraperów, obliczeń POI, modułu oceny zdjęć oraz scoringu rozmytego.

### 4.6 FRONTEND – aplikacja webowa

Frontend stanowi interfejs użytkownika systemu. Umożliwia wprowadzanie kryteriów wyszukiwania, wybór profilu użytkownika, prezentację wyników w formie listy oraz przeglądanie szczegółów pojedynczych ofert.

**Stack technologiczny:**
Aplikacja webowa została zrealizowana jako klient SPA w TypeScript z wykorzystaniem React (budowanie przez Vite). Interfejs komunikuje się z backendem przez REST.

## 5. Integracja i uruchamianie systemu

System został przygotowany do działania w dwóch trybach:

### 5.1 Tryb zdalny

Podczas synchronizacji pracy zespołu możliwe było uruchomienie usług na wybranej maszynie oraz udostępnienie ich przez **Cloudflare Tunnel**, który umożliwił zdalny dostęp do kolejki.

### 5.2 Tryb lokalny

Końcowo całość systemu została przygotowana do uruchomienia lokalnego przy użyciu **Docker Compose**. Pliki konfiguracyjne spinają wszystkie usługi, w tym backend, bazę danych, RabbitMQ, workery oraz frontend.

Kolejki wiadomości wykorzystywane są do orkiestracji pipeline’u, umożliwiając automatyczne uruchamianie kolejnych etapów przetwarzania po zakończeniu poprzednich.

## 6. Podsumowanie

M1A5TO jest modularnym systemem analizy ofert mieszkaniowych, łączącym techniki scrapowania danych, analizy grafowej i heurystycznej, uczenia maszynowego oraz logiki rozmytej. Kluczową rolę integracyjną pełni backend oparty o FastAPI i PostGIS, natomiast całość architektury została zaprojektowana z myślą o czytelnym rozdziale odpowiedzialności pomiędzy modułami.

---

