# Model popytu i scoring lokalizacji stacji ładowania EV — Polska

Oszacowanie potencjału popytu i ranking potencjalnych lokalizacji stacji/hubów
ładowania EV w Polsce, wyłącznie na podstawie danych publicznych.

## Wymagania

```bash
pip install pandas numpy folium branca matplotlib scipy requests nbformat openpyxl
```

Notebooki testowane w środowisku Python 3.11.

## Struktura folderów

```
data/         wszystkie pliki .csv, .xlsx, .geojson
Mapa/         mapa_popytu_ev.html + jej opis
notebooks/    wszystkie notatniki .ipynb
```

Notatniki uruchamiane z folderu `notebooks/`, odwołują się do danych
ścieżką `../data/`.

## Pliki wejściowe (folder `data/`)

| Plik | Skąd się bierze |
|---|---|
| `candidate_locations_FINAL_v2.csv` | Wynik wcześniejszego etapu zbierania i strukturyzacji danych (GPR, TEN-T, OSM, EIPA, CEPiK, GUS, NSP 2021) — punkt startowy tego pipeline'u |
| `1km_grid.geojson` | Siatka gęstości zaludnienia GUS 1×1 km — potrzebna do podziału popytu docelowego między lokalizacje |
| `powiaty_teryt.geojson` | Granice powiatów z kodami TERYT (Geoportal, sierpień 2025) — potrzebne do mapy i do siatki gęstości zaludnienia |

## Pipeline — kolejność uruchamiania

Uruchamiaj notebooki **w tej kolejności, od góry do dołu każdego z nich**. Każdy kolejny krok wymaga pliku wynikowego z poprzedniego.

### Krok 1 — `model_popytu.ipynb`
**Wejście:** `candidate_locations_FINAL_v2.csv`
**Wyjście:** `candidate_locations_z_popytem.csv`

Liczy szacowaną liczbę sesji ładowania i energię (kWh/rok) dla każdej z 16 232
lokalizacji, osobno dla segmentu korytarzowego (ruch drogowy) i docelowego
(lokalna flota EV). Wszystkie parametry sparametryzowane i udokumentowane
źródłowo w bloku `PARAMS` na początku notebooka.

### Krok 2 — `gęstość_zaludnienia.ipynb`
**Wejście:** `candidate_locations_z_popytem.csv`, `1km_grid.geojson`,
`powiaty_teryt.geojson`
**Wyjście:** `candidate_locations_po_korekcie_gus.csv`

Dzieli szacowany popyt segmentu docelowego między lokalizacje w tym samym
powiecie proporcjonalnie do realnej gęstości zaludnienia (siatka GUS
1×1 km), zamiast po równo. Wymaga `geopandas`.

### Krok 3 — `model_scoringowy.ipynb`
**Wejście:** `candidate_locations_po_korekcie_gus.csv`
**Wyjście:** `candidate_locations_ze_scoringiem.csv`

Dokłada do popytu: korektę pewności danych (dopasowanie ruchu drogowego),
flagę zaniżonych danych EV, premię za lukę w wymaganym rozstawie hubów
AFIR na sieci TEN-T (ważoną klasą sieci), złagodzoną korektę konkurencji,
oraz modyfikatory ruchu drogowego i ruchu dostawczego. Wynik końcowy:
`wynik_scoringowy` i `ranking_scoringowy_procentyl` — **to jest ranking, na
którym warto się opierać przy wyborze lokalizacji**, nie surowy popyt z
kroku 1.

### Krok 4 — `mapa_popyt.ipynb`
**Wejście:** `candidate_locations_ze_scoringiem.csv`, `powiaty_teryt.geojson`
**Wyjście:** `../Mapa/mapa_popytu_ev.html`

Buduje interaktywną mapę (5 warstw: mapa cieplna kandydatów, top kandydaci
korytarzowi/docelowi, istniejąca infrastruktura jako kontekst, potencjał
powiatu). Otwórz wynikowy plik `.html` w dowolnej przeglądarce.

### Krok 5 — `top800_lokalizacji.ipynb`
**Wejście:** `candidate_locations_ze_scoringiem.csv`
**Wyjście:** `top800_lokalizacji.xlsx` (2 zakładki: Korytarzowa, Docelowa)

Wybiera 800 najlepszych kandydatów pod nową inwestycję w każdym segmencie
(zdeduplikowane po nazwie+powiecie, żeby nie powtarzać tego samego węzła),
z automatycznie generowanym uzasadnieniem tekstowym dla każdej pozycji.

### Krok 6 — `eksport_xlsx.ipynb`
**Wejście:** `candidate_locations_ze_scoringiem.csv`, `top800_lokalizacji.xlsx`
**Wyjście:** `baza_lokalizacji_ev.xlsx`

Eksportuje pełną bazę lokalizacji ze scoringiem do sformatowanego pliku
Excel (4 arkusze: top 800 korytarzowe, top 800 docelowe, wszystkie
lokalizacje ze skalą kolorów, legenda kolumn).


Sprawdza, jak bardzo ranking lokalizacji zależy od trzech kluczowych założeń:
penetracji EV, skłonności do ładowania publicznego, siły konkurencji. Nie jest
wymagany do wygenerowania mapy ani XLSX — to dodatkowa, diagnostyczna analiza.

### Krok  (opcjonalny) — `walidacja_modelu.ipynb`
**Wejście:** `candidate_locations_ze_scoringiem.csv`
**Wyjście:** wydruk w konsoli, `walidacja_modelu.png`

Backtesting rankingu względem 2685 realnych hubów EIPA (moc ≥100 kW) oraz
test rozkładu wyborów trzech znanych operatorów. Nie jest wymagany do
wygenerowania mapy ani XLSX — sprawdza trafność modelu, nie generuje
danych wejściowych dla kolejnych kroków.

## Przystępne wyjaśnienie (dla osób spoza projektu)

`jak_powstala_mapa.pdf` — krótkie (3 strony), napisane bez żargonu
wyjaśnienie, skąd biorą się liczby i kolory na mapie. Nie wymaga żadnej
wcześniejszej wiedzy o projekcie.

## Pełna dokumentacja metodyczna

Wszystkie źródła danych, wzory, wartości parametrów i ich uzasadnienie:
`dokumentacja.pdf` (lub `.tex` — źródło LaTeX).

## Tabela parametrów do modeli finansowych

`tabela_parametrow.xlsx` — wszystkie parametry z obu modeli (popytu i
scoringowego), w tym rozbicie popytu bazowego na czynniki, w jednym,
sformatowanym arkuszu, gotowe do bezpośredniego wykorzystania przy
budowie modeli finansowych (utylizacja docelowa, ramp-up), bez
przeszukiwania kodu czy pełnego raportu. Plik statyczny — nie generowany
automatycznie przez żaden notebook, aktualizowany ręcznie przy zmianie
parametrów w `model_popytu.ipynb` lub `model_scoringowy.ipynb`.

## Znane ograniczenia

- Model dostarcza **ranking względny** między lokalizacjami, nie precyzyjną
  prognozę finansową w kWh/rok — patrz sekcja zastrzeżeń w dokumentacji.
- Backtesting na 2685 realnych hubach EIPA (`walidacja_modelu.ipynb`)
  pokazuje, że model dobrze różnicuje ogólną charakterystykę lokalizacji
  wybieranych przez operatorów jako grupę, ale ma ograniczoną zdolność
  przewidywania sukcesu pojedynczej stacji — szczególnie w segmencie
  korytarzowym, gdzie ranking pozostaje poniżej neutralnego poziomu mimo
  poprawek. Najlepiej traktować model jako narzędzie do rankingu
  względnego i wstępnego przesiewu, nie precyzyjną prognozę sukcesu
  konkretnej inwestycji.
- Cykliczne próbkowanie dostępności EIPA dostarcza tylko **krajowy** profil
  wykorzystania (nie per-lokalizacja) — UDT nie udostępnia publicznie
  statusu pojedynczej stacji konkurencji.
- Sanity-check modelu oparty jest na publicznie znanych sieciach (Tesla,
  GreenWay, ORLEN), nie na wewnętrznej liście lokalizacji projektu.
