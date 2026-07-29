# Jak to działa — metoda stojąca za tymi release'ami i jej wpływ na dane

*[English version](HOW-IT-WORKS.md)*

Codziennie ten pipeline publikuje, dla każdego miasta, **„zrealizowany" GTFS**: oficjalny rozkład
jazdy miasta z czasami przyjazdu i odjazdu przepisanymi tak, by odzwierciedlały to, co pojazdy
faktycznie zrobiły. Ten dokument tłumaczy, jak to powstaje, dlaczego jest zbudowane właśnie tak
i — co najważniejsze — **co te decyzje projektowe robią z liczbami, które pobierasz**.

Jest napisany dla kogoś, kto ma przed sobą zrealizowany GTFS (albo wykres różnic) i chce wiedzieć,
na ile może mu ufać i jak go czytać. To nie jest dokumentacja CLI — flagi i ich wartości domyślne
opisuje
[`tools/family_a_reconstruction/README.md`](https://github.com/GISBoost/easy-OTP/tree/main/tools/family_a_reconstruction)
w `easy-OTP`. Jak działa sama automatyzacja (harmonogramy, workflow, powiadomienia) — patrz
[`README.md`](README.md) tego repo.

**Spis treści**

1. [Jedno ograniczenie, z którego wynika wszystko](#1-jedno-ograniczenie-z-którego-wynika-wszystko)
2. [Łańcuch, krok po kroku](#2-łańcuch-krok-po-kroku)
3. [Decyzje projektowe i ich cena](#3-decyzje-projektowe-i-ich-cena)
4. [Jak czytać opublikowane liczby](#4-jak-czytać-opublikowane-liczby)
5. [Co obecnie wiadomo, że jest zepsute](#5-co-obecnie-wiadomo-że-jest-zepsute)
6. [Jak to wszystko sprawdzić samemu](#6-jak-to-wszystko-sprawdzić-samemu)

---

## 1. Jedno ograniczenie, z którego wynika wszystko

GTFS-Realtime ma dwa istotne tu typy feedów:

- **`TripUpdates`** — przewoźnik mówi wprost: *kurs X jest 4 minuty spóźniony na przystanku Y*.
- **`VehiclePositions`** — przewoźnik mówi tylko: *pojazd V jest teraz w tym punkcie, realizuje
  kurs X*.

Wiele miast publikuje wyłącznie ten drugi. **Opóźnienie nie jest więc tutaj nigdy mierzone. Jest
rekonstruowane przez wnioskowanie** — z tego, gdzie pojazdy były, kiedy, i gdzie według rozkładu
powinny być.

Ten jeden fakt jest korzeniem wszystkiego poniżej. Ponieważ opóźnienie jest wywnioskowane, a nie
zaraportowane:

- każda liczba zależy od łańcucha pośrednich decyzji (który kurs? gdzie na trasie? kiedy minął ten
  przystanek?), a **każde ogniwo może zawieść po cichu**, produkując wiarygodną liczbę zamiast
  błędu;
- wynik może opisywać wyłącznie **segmenty, które faktycznie zaobserwowano**; wszędzie indziej
  zostaje opublikowany rozkład;
- „brak korekty" i „jechał dokładnie punktualnie" wyglądają w pliku wynikowym identycznie. Tej
  dwuznaczności nie da się usunąć przy takim wejściu — §4 tłumaczy, jak sobie z nią radzić.

Metoda jest tą opisaną w Wessel, Allen & Farber (2017), *„Constructing a Routable Retrospective
Transit Timetable from a Real-time Vehicle Location Feed and GTFS"* — dopasowanie pozycji do mapy
plus interpolacja — przystosowaną do działania bez zewnętrznego silnika routingu.

---

## 2. Łańcuch, krok po kroku

Pięć kroków zamienia pingi GPS w przepisany rozkład. Każdy musi *rozstrzygnąć* coś, czego surowe
dane nie mówią wprost, i każde rozstrzygnięcie ma swoją cenę.

```
  VehiclePositions          statyczny GTFS
    (co 60 sekund)            (z tego dnia)
         |                          |
         v                          v
  1. record  ->  2. match  ->  3. kotwiczenie  ->  4. interpolacja  ->  5. agregacja
                                przystanków                                   |
                                                                              v
                                                                        6. przebudowa
                                                                    (zrealizowany GTFS)
```

### 2.1 Nagrywanie — jedno odpytanie na minutę

Telefon pracuje ciągle od ~06:00 do ~22:00 i zapisuje surowy feed raz na 60 sekund.

**Co to kosztuje:** 60 sekund to najdrobniejsza rozdzielczość czasowa, jaka może kiedykolwiek
istnieć w wyniku. Pojazd mijający przystanek pomiędzy dwoma odpytaniami nigdy nie jest przy tym
*widziany* — czas przejazdu jest interpolowany (krok 4). Gęstsze odpytywanie nie stworzyłoby więcej
obserwacji; uczyniłoby każdą z nich tylko dokładniejszą (dlaczego — patrz §2.5).

### 2.2 Dopasowanie — umieść pozycję gdzieś na trasie

Sama para współrzędnych jest bezużyteczna. Żeby coś znaczyła, musi stać się *„tyle a tyle wzdłuż
trasy R"*. Do tego potrzeba dwóch rzeczy, których pozycja sama w sobie nie niesie:

- **Który to kurs?** — brane z `trip_id` w feedzie RT. Jeśli feed tego pola nie wypełnia,
  obserwacja jest bezużyteczna. To nie jest hipotetyczne: 2026-07-20 **335 257 z 337 023 obserwacji
  Turynu nie miało `trip_id`**, a 2026-07-22 — *wszystkie* 344 449.
- **Jak wygląda trasa tego kursu?** — brane ze statycznego GTFS (`shapes.txt`). Dlatego użyty
  statyczny feed musi być **tą samą publikacją, którą feed RT właśnie serwuje**. Niektórzy
  przewoźnicy przy każdej republikacji przenumerowują całą przestrzeń `trip_id` (Łódź robi to co
  1–3 dni), więc feed statyczny, który jest jedynie *świeży*, nie wystarczy — musi być *ten
  właściwy*. Każdy dzienny release archiwizuje więc dokładnie ten statyczny GTFS, którego użyto,
  jako trzeci załącznik.

Pozycja jest następnie rzutowana na polilinię trasy, żeby uzyskać odległość wzdłuż trasy.

**Co to kosztuje:** samo rzutowanie jest niejednoznaczne wszędzie tam, gdzie trasa przechodzi blisko
samej siebie — pętla, nawrót, dwie równoległe jezdnie. Kilka metrów szumu GPS potrafi przerzucić
dopasowanie między dwoma punktami odległymi o metry w przestrzeni, ale o kilometry *wzdłuż trasy*.
Krok 2.3 oraz okienkowanie opisane w §3 istnieją po to, żeby to opanować.

### 2.3 Zakotwiczenie przystanków na tej samej linii trasy

Zanim powiesz, *kiedy* pojazd minął przystanek, musisz wiedzieć, *gdzie wzdłuż trasy ten przystanek
leży*. Brzmi trywialnie, a jest pojedynczym największym źródłem błędu w całej metodzie.

Rzutowanie każdego przystanku niezależnie na polilinię łamie się na każdej trasie, która wraca na
własną ścieżkę: przystanek blisko końca kursu potrafi zakotwiczyć się do *wcześniejszego* przejścia
przez to samo miejsce. Wynikiem jest arytmetyka wykonana poprawnie na prawdziwych danych GPS, ale
zakotwiczona w złym miejscu w przestrzeni — dając, w jednym zreprodukowanym przypadku,
„zaobserwowany" 35-minutowy czas przejazdu dla planowanego 60-sekundowego odcinka.

Stosowane są dwie strategie, w kolejności preferencji:

1. **Zaufaj pomiarom samego feedu.** Niektóre feedy publikują `shape_dist_traveled` — własną
   odległość przewoźnika wzdłuż trasy dla każdego przystanku i punktu kształtu. Tam, gdzie jest
   obecna, konsekwentnie wypełniona i spójna jednostkowo, usuwa niejednoznaczność całkowicie.
   Z monitorowanych miast tylko Praga naprawdę ją udostępnia — i to w **kilometrach**, nie metrach;
   specyfikacja nie ustala jednostki, więc trzeba ją wykrywać per feed, a nie zakładać. Łódź i Wilno
   mają tę kolumnę w plikach, ale **każda wartość jest pusta** — dlatego sprawdzenie dotyczy danych,
   a nie istnienia kolumny: *obecne* i *użyteczne* to nie to samo.
2. **Rozwiąż cały wzorzec przystanków kursu naraz.** Tam, gdzie feed tego nie dostarcza, przystanki
   są rozwiązywane *w kolejności sekwencji*, każdy przeszukując wyłącznie obszar przed poprzednim
   (z małą tolerancją wsteczną na zajezdnie i węzły przesiadkowe, gdzie realne feedy bywają
   legalnie lekko nieuporządkowane).

**Co to kosztuje:** strategia 2 to duża redukcja ryzyka, a nie dowód. Bardzo gęsto nakładające się
pętle nadal potrafią rozwiązać się błędnie, a wybór strategii zależy od tego, co dany feed akurat
publikuje — więc wyniki dwóch miast nie powstają identyczną ścieżką kodu.

### 2.4 Interpolacja — kiedy minął ten przystanek?

Mając pozycję przystanku na trasie, seria (czas, odległość) danego pojazdu jest przeszukiwana w
poszukiwaniu pary kolejnych obserwacji, która go nawiasuje, a czas przejazdu jest interpolowany
liniowo między nimi.

**Co to kosztuje — i to jest ważne:**

- Jeśli dwie nawiasujące obserwacje są odległe w czasie, interpolowany przejazd opisuje **jak rzadko
  nagrywaliśmy**, a nie jak szybko jechał pojazd. Pary odległe o więcej niż 300 sekund są więc
  odrzucane.
- Jeśli przystanek został minięty, gdy pojazd w ogóle nie był śledzony, nie ma pary nawiasującej i
  **nie powstaje żadna obserwacja**. Ten segment staje się *luką* i zachowuje rozkładowy czas.
- Pojedynczy zły odczyt GPS zanieczyszcza więcej niż jeden segment. Ponieważ każdy planowy
  przystanek, którego odległość mieści się w tej samej złej parze, dostaje tę samą błędną parę
  nawiasującą, jeden zły ping „rozlewa się". Zmierzone w Bukareszcie: **1414 anomalnych par surowych
  odczytów wyjaśniło 3613 odrzuconych obserwacji segmentów — wzmocnienie 2,56×**, bo tamtejsze
  przystanki leżą blisko siebie.

### 2.5 Agregacja — łączenie obserwacji w statystyki segmentów

Obserwacje nie są stosowane kurs po kursie. Są łączone w **klucz segmentu**:

```
(route_id, direction_id, przystanek_od, przystanek_do, day_type, kubełek_czasowy)
```

Każdy wymiar jest tam z powodu wyuczonego boleśnie:

- **trasa + kierunek + para przystanków** — naturalna jednostka „ile trwa ten odcinek".
- **day_type** (dzień roboczy / sobota / niedziela) i **kubełek czasowy** (bloki 2-godzinne) —
  dodane po tym, jak okazało się, że pojedyncze popołudniowe nagranie skorygowało **74%
  półrocznego feedu**, łącznie z kursami odjeżdżającymi o 3:48 nad ranem, których nagranie nie
  mogło zaobserwować. Bez tych dwóch wymiarów korekty przeciekają na pory i dni, dla których nie ma
  żadnych dowodów.

**Co to kosztuje — najbardziej niedoceniana własność tych danych:** jeden przejazd kursu wnosi
**co najwyżej jedną obserwację** do danego klucza, bo kurs odwiedza daną parę kolejnych przystanków
raz. Odpytywanie co minutę zagęszcza pingi *wewnątrz* kursu; nie tworzy kolejnych jego próbek.
Liczebność próby dla klucza zależy więc od tego, **jak często dana linia faktycznie jeździ w tym
2-godzinnym oknie** — autobus co godzinę wniesie najwyżej ~2 obserwacje, choćby nagranie było
idealne. Zmierzone: średnio **1,8–6,2 obserwacji na klucz**, a w Poznaniu i Pradze **28–47% kluczy
opiera się na jednej jedynej obserwacji**. Dla nich raportowana mediana *jest* tym jednym odczytem.

Każdy klucz jest następnie sprowadzany do **P50** (mediana — przypadek typowy) i **P85** (85.
percentyl — przypadek pesymistyczny). Publikowane są dwa pliki dziennie dokładnie z tego powodu:
P50 odpowiada na „ile to zwykle trwa", P85 na „ile czasu powinienem zarezerwować".

### 2.6 Przebudowa — zapisanie skorygowanego rozkładu

Na koniec rozkład jest przepisywany. Dla każdego kursu: zacznij od jego **rozkładowego pierwszego
odjazdu**, potem idź po przystankach w kolejności, używając zaobserwowanego czasu segmentu tam,
gdzie istnieje, i rozkładowego tam, gdzie nie — akumulując po drodze.

**Co to kosztuje — przeczytaj to, zanim zinterpretujesz jakąkolwiek liczbę opóźnienia.**
Raportowane opóźnienie jest **sumą bieżącą**, a nie niezależnym pomiarem per przystanek:

- jest zakotwiczone do *rozkładu* na pierwszym przystanku, więc pojazd, który wyjechał spóźniony,
  startuje z zerowym opóźnieniem z konstrukcji;
- błąd akumuluje się wzdłuż kursu, a segment **bez** obserwacji przenosi to, co narosło do tej
  pory, dalej bez zmiany, zamiast pozwolić temu wygasnąć.

Widać to wprost w opublikowanych danych. Mediana opóźnienia wg pozycji w kursie, 2026-07-24:

| Miasto | wcześnie w kursie | późno w kursie |
|---|---|---|
| Łódź | 8 s | 48 s |
| Wilno | 17 s | 67 s |
| Gdańsk | 98 s | 233 s |
| Poznań | 86 s | 147 s |
| Praga | 158 s | 244 s |

Wzrost jest monotoniczny w **każdym** zmierzonym mieście. Część tego jest prawdziwa — autobusy
faktycznie zostają coraz bardziej w tyle — ale struktura metryki gwarantuje wzrost nawet tam, gdzie
rzeczywistość by go nie dała. Przypadek Pragi jest pouczający: jej opóźnienie jest z grubsza
niezależne od trybu transportu, a **nawet metro pokazuje +158 s**, co dla systemu na zamkniętym
torowisku nie jest wiarygodne. Traktuj „opóźnienie rośnie wzdłuż kursu" jako częściowo artefakt
metody, a nie wyłącznie własność miasta.

---

## 3. Decyzje projektowe i ich cena

Każda z nich to świadomy kompromis. Prawa kolumna mówi, co to znaczy dla Ciebie.

| Decyzja | Dlaczego tak | Co to kosztuje w danych |
|---|---|---|
| Odpytywanie co 60 s | Kompromis między uprzejmością wobec feedu a precyzją; przewoźnicy limitują ruch | Czasy przejazdu są interpolowane, nigdy obserwowane wprost |
| Jeden dzień nagrania na build | Telefon nagrywa jedną ciągłą sesję na miasto na dobę | Cienkie próby na klucz segmentu (§2.5). Łączenie kilku dni jest możliwe, ale nie jest domyślne |
| Statyczny GTFS pobierany świeżo przy każdym buildzie i archiwizowany z release'em | Feedy są republikowane z przenumerowanymi `trip_id`; przestarzała statyka po cichu nie pasuje do niczego | Jeśli przewoźnik opublikuje feed *następnego* okresu z wyprzedzeniem, ten dzień degraduje się — patrz §5 |
| Klucz segmentu z day_type i kubełkiem 2-godzinnym | Powstrzymuje jedno popołudnie przed „skorygowaniem" całego feedu | 12 kubełków/dobę × 3 typy dni rozdrabnia próbę; większa rozdzielczość to mniej obserwacji na klucz |
| Minimum 2 obserwacje na segment | Pojedynczy odczyt to nie dowód | Segmenty widziane raz są odrzucane i zachowują rozkład — stają się nieodróżnialne od nieobserwowanych |
| Luka = zachowaj rozkładowy czas | Jedyny uczciwy fallback; wymyślanie liczby byłoby gorsze | **Opóźnienie dokładnie 0 jest dwuznaczne**: albo naprawdę punktualnie, albo nigdy nie zaobserwowano |
| Odrzucanie prędkości powyżej 100 km/h | Łapie artefakty interpolacji; 100 km/h jest hojne jak na miejską prędkość komunikacyjną | Legalnie szybkie połączenia też są odrzucane. W Pradze usuwa to **2187 z 123 833 kluczy**, prawie wyłącznie kolej regionalną |
| Odrzucanie par nawiasujących odległych o ponad 300 s | Taka para mierzy rzadkość nagrywania, nie prędkość | Naprawdę wolne, rzadko śledzone segmenty tracą dane razem ze złymi |
| Korekta stosuje się do każdego kursu dzielącego klucz segmentu | Jedna obserwacja na klucz to często wszystko, co jest; bez dzielenia prawie nic by się nie skorygowało | Jedna błędna obserwacja propaguje się na wszystkie kursy tej trasy/kierunku/pary/kubełka |
| P50 i P85 publikowane osobno | Mediana ukrywa ryzyko ogona; 85. percentyl to liczba do planowania | Żadna nie jest „tą właściwą"; wybierasz zależnie od zastosowania |
| Brak zewnętrznego silnika routingu | Narzędzie zostaje bez zależności i przenośne na telefon | Dopasowanie jest czysto geometryczne, bez pojęcia ciągłości trajektorii — stąd niejednoznaczność pętli z §2.3 |

---

## 4. Jak czytać opublikowane liczby

### Co znaczy tu „opóźnienie"

`opóźnienie = zrealizowany_czas_odjazdu - rozkładowy_czas_odjazdu`, dla każdego wiersza
`stop_times.txt`. Zrealizowany feed jest identyczny co do bajtu ze statycznym poza skorygowanymi
czasami, więc każdy wiersz ma swój odpowiednik, a złączenie jest dokładne.

### Zero jest dwuznaczne — i częste

`0` znaczy *albo* „jechał dokładnie według rozkładu", *albo* „nigdy nie zaobserwowano, więc został
rozkładowy czas". Nic w plikach GTFS ich nie rozróżnia. Dlatego publikowane wykresy **wykluczają
wiersze o zerowym opóźnieniu**: ich włączenie ciągnęłoby każdą średnią do zera, nic przy tym nie
znacząc.

### „% zmienionych wierszy" to nie ocena jakości

To potyka niemal każdego, więc warto powiedzieć precyzyjnie.

Nagranie obejmuje **jeden dzień**. Statyczny feed jest zwykle ważny przez **tygodnie**. Korekty są
przyjmowane tylko dla kursów, których serwis faktycznie jeździ w nagranym typie dnia — więc duża
część wierszy w ogóle nie kwalifikowała się do korekty. Uczciwym mianownikiem jest *to, co mogło
zostać skorygowane*, a nie *wszystkie wiersze*.

Zmierzone dla 2026-07-24 (piątek):

| Miasto | wiersze kwalifikujące się tego dnia | wiersze faktycznie zmienione | udział osiągalnego |
|---|---|---|---|
| Lizbona | 61,7% | 50,3% | **82%** |
| Wilno | 83,5% | 65,7% | **79%** |
| Gdańsk | 79,7% | 62,4% | **78%** |
| Praga | 68,3% | 48,0% | **70%** |
| Łódź | 35,4% | 23,8% | **67%** |

Łódź jest najlepszą ilustracją: tylko **35,4%** jej wierszy w ogóle dało się skorygować, więc
„23,8% zmienionych" to dwie trzecie maksimum — a nie porażka na dwie trzecie. **Zdrowy dzień
wypada w okolicach 66–82% osiągalnego.** Cokolwiek znacznie poniżej warto zbadać; sam surowy
procent nie mówi prawie nic.

### Opóźnienie rośnie wzdłuż kursu

Patrz §2.6. Porównując miasta albo dni, porównuj równoważne pozycje w kursach — albo przyjmij, że
linia o długich kursach zaraportuje większe opóźnienia niż linia o krótkich, niezależnie od
faktycznej punktualności.

### P50 a P85

P50 to mediana zaobserwowanego czasu segmentu; P85 to 85. percentyl, przycięty tak, by nigdy nie
był poniżej P50. Dla klucza z jedną obserwacją obie są równe tej obserwacji — plik P85 **nie**
reprezentuje wtedy dodatkowych dowodów.

---

## 5. Co obecnie wiadomo, że jest zepsute

To lista żywa, nie wyczerpująca. Sprawy są śledzone w
[issues `GISBoost/easy-OTP`](https://github.com/GISBoost/easy-OTP/issues) oraz w
[`KNOWN_ISSUES.md`](https://github.com/GISBoost/easy-OTP/blob/main/KNOWN_ISSUES.md).

### Dotyczące konkretnych miast

| Miasto | Co jest nie tak | Skutek dla danych |
|---|---|---|
| **Boston** | Buildy sprzed 2026-07-28 gubiły większość kursów (błąd wnioskowania typu na czysto numerycznych `trip_id`) | Release'y sprzed poprawki pokrywają tylko **25 ze 126 obserwowanych tras** — arbitralny, skupiskowy wycinek. **Nieporównywalne z późniejszymi buildami i nie do przeskalowania.** Naprawione na przyszłość; historyczne release'y celowo nie są przeliczane |
| **Turyn** | Jego feed `VehiclePositions` często w ogóle nie podaje `trip_id` | Dni, w których to zachodzi, produkują prawie nic. 2026-07-20 opublikował build z **217 skorygowanymi wierszami z 1 416 230**; 2026-07-22 nie dał żadnego |
| **Poznań** | Przewoźnik publikuje statykę następnego okresu kilka dni wcześniej, więc build może użyć feedu jeszcze nieważnego dla nagranego dnia | Mniej więcej **1 dzień na 3** jest mocno zdegradowany. Dwa z sześciu zbadanych dni miały feed zaczynający się *po* dacie nagrania |
| **Łódź** | **97 `shape_id` z `trips.txt` nie ma w `shapes.txt` żadnej geometrii** (18 z nich na samej linii `603`) — defekt w eksporcie samego przewoźnika | Dotknięte linie są niewidoczne: linia `603` dała **7478 obserwacji, wszystkie bezużyteczne**, a linia `R9` jest dotknięta **każdego zarchiwizowanego dnia**. Ich czasy to czysty rozkład |
| **Praga** | Płaski próg 100 km/h odrzuca legalną kolej regionalną | **2187 z 123 833 kluczy segmentu** odrzuconych, prawie wyłącznie `route_type=2`. Praska kolej jest niedokorygowana względem tramwajów i autobusów |
| **Bukareszt** | Jego surowy feed ma ~6–8× wyższy odsetek pojedynczych złych odczytów GPS niż Poznań czy Łódź (0,267% wobec 0,035–0,041% kolejnych par) | Więcej segmentów odrzuconych jako nieprawdopodobne. Filtr działa poprawnie; to wejście jest bardziej zaszumione |
| **Wilno** | Jedyne miasto bez `current_stop_sequence`, więc dopasowanie żywych pozycji opiera się wyłącznie na `stop_id` | Znana regresja skupiona na linii `A62`, której geometria źle współgra z oknem dopasowania. Odtwarzalna w każdym zbadanym dniu |

### Dotyczące wszystkich miast

- **Opóźnienie to suma bieżąca, nie pomiar per przystanek** (§2.6). Częściowo strukturalne, nie w
  pełni prawdziwe.
- **Cienkie próby.** Do ~47% kluczy segmentu opiera się na jednej obserwacji (§2.5).
- **Zerowe opóźnienie jest dwuznaczne** (§4).
- **`day_type` to lokalna data kalendarzowa, nie doba serwisowa GTFS.** Kurs nocny zaobserwowany
  tuż po północy jest przypisany do typu następnego dnia.
- **Święta nie są modelowane.** Święto z rozkładem niedzielnym jest traktowane jako ten dzień
  tygodnia, w który wypada.
- **Dwa przypadki pozostają niewyjaśnione**, oba zbadane i oba wciąż bez potwierdzonej przyczyny:
  poznańska para przystanków na Rondzie Rataje, która zostaje zawyżona po czterech osobnych
  naprawach, oraz źródło stałego offsetu opóźnienia w Pradze (akumulator z §2.6 tłumaczy *wzrost*
  wzdłuż kursu, ale nie offset obecny już na drugim przystanku).

---

## 6. Jak to wszystko sprawdzić samemu

Wszystko powyżej da się odtworzyć z publicznych artefaktów. Każdy dzienny release zawiera:

| Załącznik | Co to jest |
|---|---|
| `<miasto>_static_gtfs_<data>.zip` | Dokładnie ten statyczny feed, którego użył build |
| `<miasto>_realized_<data>_p50.zip` | Rozkład skorygowany medianą |
| `<miasto>_realized_<data>_p85.zip` | Rozkład skorygowany 85. percentylem |
| `<miasto>_diff_<data>_p50_summary.csv` | Statystyki opóźnień per trasa dla tego dnia |
| `<miasto>_diff_<data>_p50_chart.png` | Średnie opóźnienie wg pory dnia |

CSV podsumowania podaje, per `route_id`: liczbę wierszy, ile się zmieniło, `pct_changed` oraz
średnie / średnie bezwzględne / odchylenie / min / max opóźnienie w sekundach, plus wiersz `ALL`.
`pct_changed` czytaj z zastrzeżeniem o suficie z §4.

Żeby wejść głębiej — samo narzędzie rekonstrukcji jest otwarte i uruchamialne:
[`tools/family_a_reconstruction/`](https://github.com/GISBoost/easy-OTP/tree/main/tools/family_a_reconstruction)
w `easy-OTP`. `record`, `match` i `build` to osobne komendy, a `match` i `build` wypisują
diagnostykę per trasa i ostrzegają, gdy przebieg wygląda niezdrowo (trasa z obserwacjami, ale bez
ani jednej użytecznej; nieprawdopodobny poziom odrzuceń; tabela dopasowań sparowana z niewłaściwym
feedem statycznym).

Release'y przeglądasz na **[gisboost.github.io/gtfs-dashboard](https://gisboost.github.io/gtfs-dashboard/)**.

---

*Ten dokument jest na licencji [CC BY 4.0](LICENSE-docs) — cytuj, tłumacz, buduj na nim, z podaniem
autorstwa. Kod w tym repozytorium jest na [MIT](LICENSE). Dane, które dokument opisuje, nie są objęte
ani jednym, ani drugim: pochodzą z feedów poszczególnych przewoźników i podlegają ich warunkom —
zob. [Data and attribution](README.md#data-and-attribution).*
