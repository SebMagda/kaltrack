# 🟢 KalTrack — Osobisty Licznik Kalorii

> Mobilna aplikacja PWA do śledzenia kalorii i makroskładników, z AI analizą posiłków i dziennikiem wagi.  
> Zbudowana jako pojedynczy plik HTML — zero zależności, zero instalacji, działa offline.

---

## 📋 Spis treści

- [Cel aplikacji](#cel-aplikacji)
- [Stack technologiczny](#stack-technologiczny)
- [Architektura](#architektura)
- [Baza danych — schemat](#baza-danych--schemat)
- [Funkcje](#funkcje)
- [Profil użytkownika i cele kaloryczne](#profil-użytkownika-i-cele-kaloryczne)
- [Struktura plików](#struktura-plików)
- [Deployment — GitHub Pages](#deployment--github-pages)
- [Instalacja jako PWA](#instalacja-jako-pwa)
- [Rozwój projektu — roadmap](#rozwój-projektu--roadmap)

---

## Cel aplikacji

Aplikacja wspiera redukcję masy ciała poprzez:
- Śledzenie kalorii i makroskładników w czasie rzeczywistym
- AI analizę posiłków (tekst lub zdjęcie)
- Personalizowane sugestie żywieniowe po każdym posiłku
- Dziennik wagi z wykresem trendu
- Trwałe przechowywanie danych w przeglądarce (localStorage)

**Cel użytkownika:** mężczyzna, 50 lat, 108 kg, 183 cm — redukcja 10 kg tłuszczu.

---

## Stack technologiczny

| Warstwa | Technologia | Uzasadnienie |
|---|---|---|
| UI | **React 18** via `esm.sh` CDN | Reaktywny UI bez bundlera |
| Składnia | **htm 3** (tagged template literals) | JSX-like bez Babel, zero warningów |
| Styl | **Inline styles** + CSS variables | Brak zależności od plików CSS |
| Baza danych | **localStorage** (primary) | Persystuje na telefonie niezależnie od sesji |
| Backup DB | **window.storage** (Claude artifacts) | Sync gdy aplikacja działa w środowisku Claude |
| AI | **Anthropic API** `/v1/messages` | Analiza zdjęć i tekstu, sugestie dietetyka |
| Model AI | `claude-sonnet-4-20250514` | Optymalny balans jakość/koszt |
| Hosting | **GitHub Pages** | Darmowy HTTPS, wymagany dla PWA |
| Fonty | **DM Sans** via Google Fonts | Czytelny, nowoczesny |

### Dlaczego pojedynczy plik HTML?

- Zero procesu budowania (no build step)
- Łatwy deployment — przeciągnij i upuść
- Działa lokalnie (`file://`) i z serwera
- Prosta migracja między hostingami

---

## Architektura

```
kaltrack.html
│
├── <head>
│   ├── PWA meta tagi (apple-mobile-web-app-capable, theme-color)
│   ├── apple-touch-icon (SVG inline jako data URI)
│   └── CSS (reset, animacje, font import)
│
└── <script type="module">
    │
    ├── 📦 IMPORTS
    │   ├── React 18 (createElement, hooks) — esm.sh CDN
    │   ├── ReactDOM (createRoot) — esm.sh CDN
    │   └── htm — JSX-like syntax bez Babel
    │
    ├── 🗄️ DB LAYER
    │   ├── lsGet(key) — odczyt z localStorage
    │   ├── lsSet(key, val) — zapis do localStorage + sync window.storage
    │   ├── migrate() — import starych danych z window.storage (jednorazowo)
    │   └── metody domenowe: getProfile, getDayLog, getWeightLog, getAllDays...
    │
    ├── 🧩 KOMPONENTY
    │   ├── Ring — kołowy progress bar kalorii (SVG)
    │   ├── MacroBar — pasek postępu makroskładnika
    │   ├── SuggestionPanel — AI sugestie po posiłku (lazy load)
    │   ├── AddFoodModal — modal dodawania: tekst lub zdjęcie → AI → wynik
    │   ├── HistoryScreen — lista dni z paskiem postępu
    │   ├── WeightLogScreen — dziennik wagi + wykres SVG
    │   └── ProfileScreen — profil, makra, instrukcja PWA, eksport
    │
    └── 🏠 App (root)
        ├── Stan: screen, entries, profile, addingMeal, vegIdx, ready
        ├── useEffect #1 — init: migrate() + getProfile() + getDayLog(today)
        ├── useEffect #2 — autosave: setDayLog() przy każdej zmianie entries
        └── Router: if(screen==='history') return <HistoryScreen/> ...
```

### Przepływ danych — dodawanie posiłku

```
Użytkownik tapnij "+"
    ↓
AddFoodModal otwiera się
    ↓
Tryb TEXT: wpisuje nazwę + opcjonalnie gramaturę
Tryb PHOTO: robi zdjęcie → FileReader → base64
    ↓
fetch POST → api.anthropic.com/v1/messages
  payload: { model, messages: [{ role, content: [image?, text] }] }
    ↓
Odpowiedź JSON: { name, calories, protein, carbs, fat, confidence }
    ↓
Użytkownik potwierdza → onAdd(food)
    ↓
setEntries(prev => [...prev, { ...food, mealId }])
    ↓
useEffect → DB.setDayLog(today, entries) → localStorage
```

### Przepływ AI sugestii

```
Użytkownik tapnij "Co powinienem teraz zjeść?"
    ↓
Zbiera kontekst:
  - zjedzone w tym posiłku (nazwy + kcal)
  - łączne kalorie posiłku + białko
  - pozostałe kcal na dziś
  - informacje o następnym posiłku
    ↓
fetch POST → Anthropic API (prompt z profilem dietetycznym)
    ↓
JSON: { ocena, emoji, dojedz[], nastepny[], warzywa[] }
    ↓
Render w SuggestionPanel
```

---

## Baza danych — schemat

Wszystkie klucze mają prefix `kal:` w localStorage.

```
localStorage
│
├── kal:profile
│   └── { weight: 108, height: 183, age: 50, dailyGoal: 2400, tdee: 3067 }
│
├── kal:day:YYYY-MM-DD           (np. kal:day:2025-04-20)
│   └── {
│         date: "2025-04-20",
│         entries: [
│           {
│             id: 1713600000000,    // Date.now() jako unikalny ID
│             mealId: "breakfast",  // "breakfast" | "lunch" | "dinner"
│             name: "Jajecznica z 3 jajek",
│             calories: 280,
│             protein: 18,          // gramy
│             carbs: 2,
│             fat: 22,
│             weight: 150,          // gramy porcji (opcjonalne)
│             time: "07:15"         // godzina dodania
│           },
│           ...
│         ],
│         updatedAt: 1713600000000
│       }
│
├── kal:weight_log
│   └── [
│         { date: "2025-04-20", weight: 107.5, ts: 1713600000000 },
│         { date: "2025-04-13", weight: 108.0, ts: 1713000000000 },
│         ...                        // posortowane malejąco po dacie
│       ]
│
└── kal:_v2                          // flaga migracji (string "1")
```

### Zasady DB

- **Klucze** — zawsze z prefixem `kal:` aby nie kolizjonować z innymi aplikacjami
- **ID wpisów** — `Date.now()` — wystarczająco unikalny w kontekście jednego użytkownika
- **Sortowanie wagi** — malejąco po dacie, aktualizacja in-place jeśli data już istnieje
- **Autosave** — `useEffect` reaguje na każdą zmianę `entries`, zapis async w tle
- **Backup** — `window.storage` (Claude) synchronizowany przy każdym zapisie, ale nie jest primary

---

## Funkcje

### Ekran główny (Dziś)
- Kołowy licznik kalorii z animacją (SVG)
- Paski postępu: Białko / Węglowodany / Tłuszcze vs. cele dzienne
- 3 sekcje posiłków z indywidualnymi celami kalorycznymi
- Rotating tip o warzywach (wypełniacz, zero kalorii)
- Zielony wskaźnik statusu localStorage

### Dodawanie posiłku
- **Tryb tekstowy** — wpisz nazwę + opcjonalnie gramaturę → AI zwraca kalorie i makra
- **Tryb zdjęcia** — camera/galeria → base64 → AI rozpoznaje potrawę
- Wynik zawiera: nazwa, kcal, białko, węgle, tłuszcz, poziom pewności AI
- Enter key support w polach tekstowych

### AI Sugestie po posiłku
- Ocena posiłku (✅/⚠️/❌) z komentarzem
- "Dojedz teraz" — brakujące składniki z uzasadnieniem
- Propozycje na następny posiłek z makrami
- Warzywa do jedzenia bez limitu (zawsze obecne)
- Lazy loading — wywołuje API tylko przy tapnięciu

### Historia
- Lista wszystkich zalogowanych dni
- Pasek postępu kcal vs. cel
- Informacja o deficycie lub nadwyżce
- Łączny deficyt ze wszystkich dni

### Dziennik wagi
- Dodawanie pomiaru (data + waga)
- Wykres trendu SVG (ostatnie 14 pomiarów)
- Statystyki: waga startowa / aktualna / strata
- Usuwanie wpisów

### Profil
- Instrukcja instalacji PWA (iPhone + Android)
- Makroskładniki dzienne (białko 180g / węgle 270g / tłuszcze 67g)
- Dane profilu i cele
- Zasady diety
- Eksport wszystkich danych do JSON (backup)

---

## Profil użytkownika i cele kaloryczne

### Obliczenia BMR / TDEE

Wzór Mifflin-St Jeor dla mężczyzn:
```
BMR = 10 × waga(kg) + 6.25 × wzrost(cm) - 5 × wiek + 5
BMR = 10×108 + 6.25×183 - 5×50 + 5 = 1 979 kcal

TDEE = BMR × 1.55 (umiarkowanie aktywny, 3-4x sport/tydzień)
TDEE = 1 979 × 1.55 ≈ 3 067 kcal
```

### Cel kaloryczny

| Parametr | Wartość |
|---|---|
| TDEE | 3 067 kcal |
| Deficyt | 667 kcal/dzień |
| **Cel dzienny** | **2 400 kcal** |
| Tempo redukcji | ~0,75 kg/tydzień |
| Czas do celu (10 kg) | ~13 tygodni |

### Rozkład makroskładników

| Makro | Gramów | % kalorii | Kalorie |
|---|---|---|---|
| Białko | 180 g | 30% | 720 kcal |
| Węglowodany | 270 g | 45% | 1 080 kcal |
| Tłuszcze | 67 g | 25% | 600 kcal |

### Harmonogram posiłków

| Posiłek | Godzina | Cel kcal |
|---|---|---|
| 🌅 Śniadanie | 7:00 | 600 kcal |
| ☀️ Obiad | 14:30 | 960 kcal |
| 🌙 Kolacja | 19:30 | 840 kcal |

---

## Struktura plików

```
kaltrack/
├── kaltrack.html          # Cała aplikacja (single-file PWA)
└── README.md              # Ta dokumentacja
```

Celowo minimalistyczna struktura — jedna zmiana w jednym pliku, jeden commit.

---

## Deployment — GitHub Pages

### Jednorazowa konfiguracja

1. Utwórz repozytorium `kaltrack` na GitHub (Public)
2. Wgraj `kaltrack.html` i `README.md`
3. Settings → Pages → Branch: `main`, folder: `/ (root)` → Save

### URL aplikacji

```
https://<twoja-nazwa>.github.io/kaltrack/kaltrack.html
```

### Aktualizacja aplikacji

```bash
# Edytuj kaltrack.html lokalnie, potem:
git add kaltrack.html
git commit -m "feat: opis zmiany"
git push
# GitHub Pages automatycznie wdraża po ~1 minucie
```

> ⚠️ **Dane są bezpieczne przy aktualizacjach** — localStorage jest powiązany z domeną (`github.io`), nie z zawartością pliku. Aktualizacja kodu nie kasuje danych użytkownika.

---

## Instalacja jako PWA

### iPhone / iPad (Safari)
1. Otwórz URL aplikacji w **Safari** (nie Chrome)
2. Tapnij ikonę **Udostępnij** (kwadrat ze strzałką ↑)
3. Wybierz **"Dodaj do ekranu głównego"**
4. Potwierdź nazwę → **"Dodaj"**

### Android (Chrome)
1. Otwórz URL w **Chrome**
2. Menu **⋮** → **"Dodaj do ekranu głównego"**
   lub banner "Zainstaluj aplikację" jeśli się pojawi
3. Potwierdź → ikona na pulpicie

> Aplikacja ma skonfigurowane meta tagi PWA: `apple-mobile-web-app-capable`, `theme-color: #080f08`, `apple-touch-icon` (SVG jako data URI).

---

## Rozwój projektu — roadmap

### Planowane funkcje

- [ ] **Analiza tygodniowa** — wykres kalorii z 7 dni, średnia deficytu
- [ ] **Baza produktów** — często używane produkty, szybkie dodawanie
- [ ] **Przypomnienia** — Web Notifications API dla godzin posiłków
- [ ] **Pomiary ciała** — obwód talii, bioder, klatki piersiowej
- [ ] **Import JSON** — przywracanie danych z backupu
- [ ] **Edycja profilu** — zmiana celu kalorycznego w aplikacji
- [ ] **Tryb jasny** — alternatywny motyw kolorystyczny
- [ ] **Eksport CSV** — dla zewnętrznych narzędzi (Excel, Google Sheets)

### Znane ograniczenia

- AI wymaga połączenia z internetem (Anthropic API)
- Brak synchronizacji między urządzeniami (dane tylko lokalnie)
- Kalorie ze zdjęć są szacunkowe (poziom pewności AI: wysoki/średni/niski)
- localStorage limit: ~5–10 MB (wystarczy na lata logowania)

---

## Licencja

Projekt prywatny. Kod stworzony przy pomocy Claude (Anthropic).
