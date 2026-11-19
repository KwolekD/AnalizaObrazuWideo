# Lab 4: kontekst dla nowych sesji

Stan na 2026-11-19 (krok 6 przeniesiony do `detect_objects`). Odpowiedzi i markdown w notatnikach piszemy po polsku.

## Co jest w katalogu

| plik | co to |
|---|---|
| `Lab4_detekcja.ipynb` | część 1: detekcja i segmentacja larw mącznika + analiza instancji (długość, grubość, kształt I/C/S). **Gotowa**, główny przedmiot pracy. |
| `Lab4_Hough.ipynb` | część 2: bile i bandy snookera transformacją Hougha. Ma podsumowanie (sekcja 6); w sesji, w której powstał ten plik, nie była ruszana. |
| `Lab4_teoria/Amodal_...pdf` | artykuł Dolata i in., IEEE Access 2025. Kroki 1–5 potoku z części 1 odwzorowują sekcję **III-B2c** (segmentacja instancji). Krok 6 to rozszerzenie. |
| `Lab4_teoria/*.ipynb`, `Lab4_teoria_Hough.pdf` | materiały prowadzącego (morfologia, momenty, Hough). |
| `Lab4_dane/` | `proc_*112112426*` = `calib` (dobór parametrów), `proc_*112219030*` = `test`; `mcgill*`, `trump`, `zhao` = część 2. |
| `notatnik` | ustne wskazówki prowadzącego: kontur + bbox na instancję, odfiltrować stykające się larwy, funkcja wizualizacji, zacząć od łatwego RoI, opisać badanie krok po kroku, wyjaśnić, dlaczego coś się zgubiło. |

## Uruchamianie i sprawdzanie

- Python tylko z `.venv` w katalogu głównym repo: `.venv/Scripts/python.exe`. Systemowego `python` nie ma w PATH.
- Wykonanie całości: `.venv/Scripts/jupyter-nbconvert.exe --to notebook --execute --inplace lab_4/Lab4_detekcja.ipynb`. Trwa ok. 100 s (przebiegi progu globalnego i `alpha` liczą też krok 6). Ścieżki do danych są względne wobec `lab_4/`, a nbconvert sam ustawia ten katalog.
- Notatnik z wyjściami ma ok. 36 MB. Nie czytaj go narzędziem Read: źródła i wyjścia tekstowe wyciągaj skryptem (`json.load` → `cells[i]['source']`, `outputs`). Indeksy komórek się zmieniają, więc szukaj komórek po początku źródła.
- Oczekiwane wyniki po wykonaniu:
  - `S_THR = 104`,
  - `calib`: 86 larw po kroku 5 → **92** po kroku 6, odrzucone ostatecznie `{szkielet: 10, dlugosc: 2, grubosc: 2}`, 289 drobnych obiektów,
  - `test`: 90 larw po kroku 5 → **103** po kroku 6, odrzucone ostatecznie `{szkielet: 12, dlugosc: 4, grubosc: 1}`, 294 drobne obiekty,
  - RoI: 8 larw (krok 6 nic nie zmienia),
  - krok 6: `calib` zlepki 3 z 17 (+6 larw), wypustki usunięte u 4 z 10 larw; `test` zlepki 7 z 24 (+13 larw, w tym @(3319, 2230) zamieniony na jedną larwę bez okruchu), wypustki usunięte u 3 z 8,
  - eksperyment `max_extra`: pierwsza reguła odrzuciłaby 10 z 86 larw (było 11, dopóki galeria liczyła szkielet bez marginesu `FEAT_PAD`).
- D (przecięcie histogramów) i mediany z „Wniosków z analizy” są tylko w tytułach i legendach wykresów (komórka „Statystyki populacji”), nie w `print`. Sprawdzaj je, wyciągając PNG z wyjścia. Obecnie: D = 0.85 (długość) i 0.85 (grubość), mediany długości 161.4 / 159.4 px.
- Użytkownik zwykle ma notatnik otwarty w VS Code. Przed nadpisaniem pliku zrób kopię w scratchpadzie, a po nadpisaniu przypomnij o „Revert File”.

## Budowa `Lab4_detekcja.ipynb`

Każdy krok potoku to osobna funkcja i osobna sekcja w układzie: opis → wynik na łatwym RoI → eksperyment → **Wniosek**. Funkcje wszystkich kroków są w komórce zaczynającej się od `K_GROW = ...`:

| krok | funkcja | co robi |
|---|---|---|
| 1 | `rough_mask` | `S > S_THR` (kanał S z HSV) |
| 2 | `find_candidates` | `findContours`, odrzuca obiekty `< min_area`, wyznacza ROI z marginesem |
| 3–4 | `segment_candidate` | lokalny Otsu ograniczony do konturu wstępnego poszerzonego o `K_GROW`, potem erozja `refine_edges` na kanale B z `delta` = `alpha` × `b_contrast` (mediana B tła ≥ 4 px od maski − średnia B obiektu), na koniec największa składowa nakładająca się na kontur, z wypełnionymi dziurami |
| 5 | `smooth_skeleton`, `blob_features` + `filter_candidates` | szkielet maski po otwarciu `K_SMOOTH` (z marginesem `FEAT_PAD`): najdłuższa ścieżka, nadmiar gałęzi, grubość z transformaty odległości; potem powody odrzucenia. `smooth_skeleton` jest wspólny dla `blob_features` i wizualizacji (`skeleton_nodes`), więc galeria pokazuje ten sam szkielet, z którego liczony jest nadmiar |
| 6 | `split_candidates` → `split_candidate` → `clump_parts` | dla zlepków odrzuconych w kroku 5 (`CLUMP_REASONS`) i przyjętych larw z boczną gałęzią (nadmiar ≥ `branch_extra`): wyższy próg na S w obrębie maski, otwarcie, znaczniki ≥ `min_part`, odbudowa `expand_labels`, najniższy próg z poprawnym podziałem. Zwraca listę `dict(cand, thr, parts, kind)`, `kind` = `'zlepek'` / `'wypustka'` |
| całość | `detect_objects` | wywołuje kroki 1–6; domyślne argumenty = **parametry końcowe**; `split=False` daje wynik kroków 1–5, `**split_kw` trafia do `split_candidates`. `debug=True` zwraca też `info`: `candidates` (po kroku 5, z obrazami pośrednimi), `splits` (krok 6), `filt` (parametry i mediany filtra), `rejected` (ostatecznie odrzuceni) |

Krok 6 jest w głównym potoku: decyzja użytkownika z 2026-11-19 („to larwy, które da się oddzielić, więc czemu nie poprawić wyniku”). Wcześniej był dodatkową analizą poza `detect_objects`. Analiza instancji liczy się na pełnym wyniku (`objs_calib`, `objs_test`).

Na co uważać przy zmianach:
- `filter_candidates` nie zmienia wejścia i działa na gotowych cechach. Eksperymenty z progami filtra rób przez nią, bez ponownej segmentacji.
- Krok 6 też można liczyć bez ponownej segmentacji: `split_candidates(info['S'], info['candidates'], info['filt'], **kw)`. Tak działa ablacja kroku 6.
- `blob_features` dokleja margines `FEAT_PAD = 10`, żeby szkielet obiektów przy brzegu liczył się poprawnie.
- `detect_objects` zachowuje kolejność kandydatów i wstawia części z kroku 6 w miejsce kandydata, z którego powstały.
- `show_candidate_steps` pokazuje decyzję kroku 5, więc przykłady „krok po kroku” bierz z kandydatów, których krok 6 nie zmienił.

Kolejność sekcji:
1. obszar roboczy (przycięcie `FLOOR`),
2. „Potok detekcji”: jedna krótka tabela (krok → funkcja → artykuł → parametry końcowe), bez opisów kroków, bo te są w sekcjach kroków. Osobnej sekcji „Odstępstwa od artykułu” już nie ma (usunięta przez użytkownika), odstępstwa są opisane w sekcjach kroków,
3. łatwy RoI,
4. kroki 1 / 2 / 3–4 / 5 z eksperymentami (pełna detekcja `objs_*`, `dbg_*` jest liczona w sekcji kroku 5),
5. krok 6: wynik (galerie zlepków i wypustek), eksperyment „czy wypustki usuwa binaryzacja albo erozja B”, ablacja po jednym parametrze z galerią podziałów przyjmowanych tylko bez zabezpieczeń,
6. wyniki dla `calib` i `test` (z analizą strat),
7. wrażliwość wyniku na próg globalny, skrócona na prośbę użytkownika: tabela i wykres dla progów 60–160 oraz krótki wniosek,
8. analiza instancji: `analyze_instance` / `analyze_objects`, walidacja na syntetycznych larwach, eksperyment z erozją a pomiarem.

## Decyzje z uzasadnieniem (nie zmieniać bez nowego eksperymentu)

- **`FLOOR` = (255:2730, 132:3970)**, czyli dno pudełka (275:2710, 152:3950) poszerzone o 20 px z każdej strony.
  - Wycina ścianki i niebieską taśmę, która ma wysokie S.
  - Margines jest po to, żeby nie tracić larw leżących przy ściance (+1 larwa na każdym obrazie, 0 zgubionych, 0 fałszywych obiektów ze ścianek).
  - Dla marginesu 20–60 px wynik jest identyczny. 20 px to najmniejsza wartość, przy której żaden obiekt na `calib` ani `test` nie dotyka brzegu, więc `krawedz` działa już tylko na RoI.
  - Larwa wspinająca się po lewej ściance `test` @(3, 82) odpada przez `grubosc` (1.32×), bo maska łapie cień na ściance. Krok 6 zdejmuje cień przy progu 180, ale zostaje 92% maski, mniej niż `min_cover`.
  - Wszystkie współrzędne „w układzie przyciętego obrazu” (RoI, powiększenia, @(x, y) w markdownie) są liczone od tego poszerzonego prostokąta.
- **`S_THR` = Otsu na `calib` = 104.** Dla progów 90–110 wynik jest identyczny poza bladą larwą `calib` @(3546, 1129), która przy 100 i 110 odpada (`grubosc` albo `szkielet`). Z krokiem 6 wyższy próg globalny prawie nic nie daje (bilans przy 120–160: 0 do +2 na `calib`, −1 do +1 na `test`).
- **`min_area` = 1500.** Leży w pustej przerwie rozkładu pól komponentów (ok. 960 → 1730 px).
- **`K_GROW` (ograniczenie lokalnego progu).** Przy 104 nie zmienia wyniku, ale przy wyższym progu globalnym jest konieczne: bez niego przy 130 na `test` przybywają 2 larwy zamiast 6, bo lokalny Otsu (ok. 110) z powrotem skleja larwy rozdzielone w masce wstępnej (pomiar sprzed przeniesienia kroku 6 do potoku).
- **Usunięte: dwa otwarcia 3×3** (po progu globalnym i po erozji). Przy progach 70–110 nic nie zmieniały.
- **`refine_alpha` = 0.25, `refine_iters` = 4** (wcześniej stała `refine_delta` = 30; zmiana na prośbę użytkownika, 2026-11-19).
  - Kontrast B tło − larwa w ROI to 62–170, bo pudełko jest nierówno oświetlone. Stała `delta` = 30 to od 0.18 do 0.48 kontrastu, więc na jasnym tle erodowała mocniej.
  - 0.25 daje `delta` ≈ 30 dla mediany kontrastu i średnio tyle samo zdjętych pikseli (5.6% / 5.1%). Krzywa „zdjęto(alpha)” nie ma kolana. Przy 0.15 erozja wcina się w ciało po stronie oświetlonej. Liczba larw (z krokiem 6) dla 0.10–0.50: 90–92 / 102–103.
  - Względem stałej `delta` zmieniły się dwie decyzje: blada larwa `calib` @(3546, 1129) jest przyjęta (1.48× → 1.19× grubości, maska z wcięciami, niestabilna względem `S_THR`), a para `test` @(707, 728) nie jest już rozcinana erozją. Odpada jako `szkielet` i rozdziela ją krok 6 (obie larwy).
  - Zgodność rozkładu grubości `calib`/`test` wzrosła z D = 0.79 do 0.85.
  - **Decyzje użytkownika:** `alpha` zostaje 0.25. Blada, gruba larwa `calib` @(3546, 1129) jest OK i zostaje przyjęta, mimo wcięć w masce, odstającego punktu w analizie (ok. 110 × 31 px) i niestabilności względem `S_THR`. Parę `test` @(707, 728) pokazujemy jako przykład zlepka, który rozdziela krok 6 (próg 120, obie larwy). Nie wracać do stałej `delta`.
  - Kształt jądra (krzyż 3×3, elipsa 5×5, okno średniej 5×5) sprawdzony skryptem poza notatnikiem: praktycznie bez wpływu, bo o tym, co znika, decyduje warunek na B.
  - Erozja zmienia głównie grubość (22.5 px bez erozji, 19.75 px przy 0.15), a liczbę larw prawie wcale.
- **`max_extra` = 30 px.** Poprzednia reguła „2 końce, 0 rozgałęzień” odrzucała larwy z drobną wypustką.
- **`k_len` = 1.35** (wcześniej 1.5).
  - Pojedyncze larwy mają do 1.25 × mediany, zlepki od 1.47 ×.
  - Próg 1.5 przepuszczał parę stykającą się końcami: `calib` @(3266, 1186).
  - Dla `k_len` od 1.26 do 1.46 liczba przyjętych larw się nie zmienia.
- **`k_width` = 1.25.** Pojedyncze larwy do 1.23 ×, zlepki od 1.26 ×; przerwa jest wąska.
- **`k_area` = 1.6.** Nigdy nie decyduje jako pierwszy, to tylko zabezpieczenie. Samo pole nie oddziela pojedynczych larw od zlepków.
- **Próg 130 sprawdzony i odrzucony** (szczegółowa analiza usunięta z notatnika, zastępuje ją krok 6).
  - Bez kroku 6 dawał 7 nowych larw na `test`: 4 dobre, 1 wątpliwą, 2 ze złą maską. Z krokiem 6 daje tylko 2 nowe i gubi 1.
  - Zasada przy budowie bazy instancji: lepiej zgubić larwę niż zapisać złą maskę.
- **Wypustki w masce (pytanie użytkownika, 2026-11-19).** Przyjęte larwy mają nadmiar 0 albo ≥ 4 px (10 na `calib`, 8 na `test`). Wypustki to głównie okruchy, kawałki wylinki i odchody, czasem odnóże albo zagięta główka.
  - Binaryzacja globalna ani mocniejsza erozja B ich nie usuwa: piksele usunięte przez krok 6 mają S ≈ 154 i B ≈ 53, ciało S ≈ 183 i B ≈ 45, a próg lokalny to ok. 100–115. Przy `alpha` = 0.15 nadmiar znika u 7 z 18 larw, ale u 7 rośnie, a blada larwa odpada.
  - Otwarcie kształtem (elipsa 9–15 + największa składowa + odbudowa) sprawdzone skryptem poza notatnikiem: prawie bez efektu, bo odbudowa przywraca krótkie wypustki. Wariant z blokującymi drobnymi znacznikami i szerszą odbudową (10–25 px) też gorszy.
  - Dlatego wypustki usuwa krok 6 (lokalnie wyższy próg). Zostaje 6 + 5 larw z wypustką. U 6 gałąź nie znika do progu 200, a u 5 znika dopiero przy 180–190, gdy zostaje 84–91% maski.
- **Krok 6, parametry** (ablacja w notatniku, wszystkie przyjęte zmiany sprawdzone wzrokowo):
  - progi co 10 (`thr_step`), od pierwszego powyżej progu lokalnego kandydata do 200. Krok progu 5 daje te same podziały (część o 5 niżej) i dodatkowo oczyszcza bladą larwę `test` @(632, 1251). Nie przyjęte, do decyzji użytkownika.
  - `open_k` = 5: przy 0 i 3 ginie para `calib` @(2835, 1810), przy 7 pary `test` @(2438, 1700) i @(2770, 849). Inne wartości oczyszczają trochę inne larwy.
  - `grow` = 6: bez odbudowy prawie nic nie przechodzi, przy 3 px −1 / −2 zlepki, przy 10 px te same podziały zlepków, ale −5 / +2 oczyszczeń (odbudowa dokleja wypustki).
  - `min_part` = 500: zakres 200–1000 bez wpływu.
  - `k_min` = 0.7: blokuje tylko oczyszczenie krótkiej bladej larwy @(3546, 1129) (0.58×). Zły podział `test` @(393, 1214) (0.49×) ma pokrycie 0.937, więc blokuje go też `min_cover`.
  - `min_cover` = 0.94 (wcześniej 0.9, zmiana przy rozszerzeniu kroku 6 na wypustki). Przyjęte zmiany ≥ 0.951, odrzucone ≤ 0.921 (`calib` @(1632, 253) przy 0.907 ucina własny ogon, larwa na ściance `test` @(3, 82) 0.921). Wynik stały dla 0.93–0.95. Koszt: `calib` @(1032, 172) (0.50) i `test` @(1797, 1123) z dużą wylinką (0.82).
  - `clean_extra` = 3 px (wynik z jedną częścią musi być bez bocznej gałęzi), `branch_extra` = 3 px (które przyjęte larwy idą do kroku 6). Bez warunku krok 6 „oczyszcza”, zostawiając gałąź, i przyjmuje `calib` @(1509, 339) z wylinką (nadmiar 25).
  - Wersja pierwsza (stały wysoki próg, bez zabezpieczeń) przyjmowała na `test` ok. połowę złych masek (kawałki larw przykrytych przez inne).

## Konwencje pracy (potwierdzone z użytkownikiem)

- **Eksperymenty:** liczone przy parametrach końcowych, każdy zmienia jeden parametr (styl ablacji). Notatnik mówi wprost, że parametry dobierano iteracyjnie. Jeśli efekt kroku znika w dalszej części potoku, pokazuj go na wyniku etapu i na wyniku końcowym.
- **Ablacja w całym zakresie:** sprawdzaj operację w całym zakresie parametrów, nie tylko w punkcie pracy. Ablacja `K_GROW` wyłącznie przy 104 dała błędny wniosek.
- **Biblioteki:** używaj gotowych funkcji OpenCV / skimage / scipy. Własny kod pisz tylko tam, gdzie biblioteki nie mają odpowiednika, np. przy adaptacyjnej erozji z artykułu.
- **Styl markdownu:** pierwsza osoba („dobieram”, „sprawdzam”), zwięźle, wnioski oparte na liczbach.
- **Nazewnictwo:** krok 1 to „binaryzacja wstępna”, a jego wynik to „maska wstępna” / „kontur wstępny” (w kodzie `rough_mask`, `rough`). Nie wracać do „maski zgrubnej”.
- **Liczby w markdownie są wpisane ręcznie.** Po każdej zmianie parametru wykonaj notatnik, porównaj wyjścia tekstowe przed i po, i popraw wszystkie liczby, które się zmieniły. Chodzi o wnioski z kroków 1, 2, 3–4, 5 i 6, tabelę „Wyniki detekcji i dlaczego część larw się zgubiła”, wnioski o wrażliwości na próg, wnioski z eksperymentu z erozją i „Wnioski z analizy”.
- Eksperyment z kroku 1 (próg a maska wstępna) nie ma już `print` (usunął go użytkownik). Liczby z wniosku kroku 1 widać tylko na wykresach, więc sprawdzaj je skryptem.
- **Kolory decyzji** (prośba użytkownika, 2026-11-19): na wyniku końcowym w każdej wizualizacji zielony kontur = przyjęty, czerwony = odrzucony. Stałe `ACCEPT_RGB` / `REJECT_RGB` są w komórce z `display_objects`. Czerwieni i zieleni nie używać do niczego innego przy konturach i nakładkach: ścieżka szkieletu jest niebieska, wypełnienia instancji losowane z odcieni cyjan–fiolet, części w kroku 6 rozróżnia wypełnienie (`SPLIT_FILLS`), biały kontur to maska z kroku 5, a kontur części ma kolor decyzji (`show_splits(..., accepted=False)` dla podziałów odrzuconych). Wyjątki zostawione świadomie: `show_erosion` (zieleń = maska po erozji, prośba użytkownika) i klasy kształtu w analizie instancji (C zielone, S czerwone, to linia środkowa, nie kontur).
- **Wizualizacja erozji** (prośba użytkownika): maska po erozji jako półprzezroczysta zieleń, kontur maski z kroku 3 na niebiesko (`find_contours(local, 0.5)`, czyli po granicy pikseli). Pas między nimi to piksele zdjęte. Nie zaznaczać usuniętych pikseli magentą. Funkcja `show_erosion` jest w komórce eksperymentu kroków 3–4. `show_candidate_steps` ma wciąż panel z magentą; jego ostatni panel ma kontur w kolorze decyzji.
- **Galeria pod rozkładem nadmiaru** (`show_skeleton_vs_final`, prośba użytkownika): dwa panele na larwę. Lewy: pierwsza reguła (biały kontur maski po otwarciu, szkielet, końce, rozgałęzienia liczone jako skupiska pikseli). Prawy: wynik końcowy, czyli maska z wyników (po kroku 6, jeśli ten ją zmienił; wtedy biała przerywana linia = maska z kroku 5), kontur decyzji, najdłuższa ścieżka (niebieska) i gałęzie nadmiaru (pomarańczowe).

## Otwarte wątki

- `min_cover` nie rozróżnia „zniknęła larwa leżąca obok” / „odpadł duży okruch” od „zniknęła larwa, której kawałek dokleił się do sąsiadki”, dlatego gubi 2 dobre larwy w `calib` @(1032, 172) i jedną w `test` @(1797, 1123).
- 11 przyjętych larw zostaje z wypustką w masce (krok 6 ich nie oczyszcza). Przy `calib` @(1863, 647) najdłuższa ścieżka skręca w ciemny wyrostek zamiast iść do końca larwy, więc długość i kształt tej larwy mogą być przekłamane. Niezbadane.
- `thr_step` = 5 dałoby +1 oczyszczoną larwę (`test` @(632, 1251)), ale zmienia progi kilku zweryfikowanych podziałów. Nie przyjęte.
- Reguła „największa składowa” gubiła larwę `test` @(795, 729). Przy `alpha` = 0.25 cała para @(707, 728) trafia do kroku 6, który odzyskuje obie larwy.
- Znane straty:
  - ciemna larwa w `calib` ok. @(3308, 1716) ma S ≈ 98, czyli tuż pod progiem,
  - jasne larwy tuż po wylince mają niskie S.
- Brak skali na zdjęciach, więc wszystkie pomiary są w pikselach.
