# Lab 4: kontekst dla nowych sesji

Stan na 2026-11-19 (commit `ccfc1ed`). Odpowiedzi i markdown w notatnikach piszemy po polsku.

## Co jest w katalogu

| plik | co to |
|---|---|
| `Lab4_detekcja.ipynb` | część 1: detekcja i segmentacja larw mącznika + analiza instancji (długość, grubość, kształt I/C/S). **Gotowa**, główny przedmiot pracy. |
| `Lab4_Hough.ipynb` | część 2: bile i bandy snookera transformacją Hougha. Ma podsumowanie (sekcja 6); w sesji, w której powstał ten plik, nie była ruszana. |
| `Lab4_teoria/Amodal_...pdf` | artykuł Dolata i in., IEEE Access 2025. Potok z części 1 odwzorowuje sekcję **III-B2c** (segmentacja instancji). |
| `Lab4_teoria/*.ipynb`, `Lab4_teoria_Hough.pdf` | materiały prowadzącego (morfologia, momenty, Hough). |
| `Lab4_dane/` | `proc_*112112426*` = `calib` (dobór parametrów), `proc_*112219030*` = `test`; `mcgill*`, `trump`, `zhao` = część 2. |
| `notatnik` | ustne wskazówki prowadzącego: kontur + bbox na instancję, odfiltrować stykające się larwy, funkcja wizualizacji, zacząć od łatwego RoI, opisać badanie krok po kroku, wyjaśnić, dlaczego coś się zgubiło. |

## Uruchamianie i sprawdzanie

- Python tylko z `.venv` w katalogu głównym repo: `.venv/Scripts/python.exe`. Systemowego `python` nie ma w PATH.
- Wykonanie całości: `.venv/Scripts/jupyter-nbconvert.exe --to notebook --execute --inplace lab_4/Lab4_detekcja.ipynb`. Trwa ok. 55 s. Ścieżki do danych są względne wobec `lab_4/`, a nbconvert sam ustawia ten katalog.
- Notatnik z wyjściami ma ok. 29 MB. Nie czytaj go narzędziem Read: źródła i wyjścia tekstowe wyciągaj skryptem (`json.load` → `cells[i]['source']`, `outputs`).
- Oczekiwane wyniki po wykonaniu:
  - `S_THR = 104`,
  - `calib`: 86 larw, odrzucone `{szkielet: 11, dlugosc: 4, grubosc: 2}`, 289 drobnych obiektów,
  - `test`: 90 larw, odrzucone `{szkielet: 17, dlugosc: 6, grubosc: 1}`, 294 drobne obiekty,
  - RoI: 8 larw,
  - krok 6: `calib` rozdzielone 3 z 17 zlepków (+6 larw), `test` 6 z 24 (+12 larw).
- D (przecięcie histogramów) i mediany z „Wniosków z analizy” są tylko w tytułach i legendach wykresów (komórka „Statystyki populacji”), nie w `print`. Sprawdzaj je, wyciągając PNG z wyjścia.
- Użytkownik zwykle ma notatnik otwarty w VS Code. Przed nadpisaniem pliku zrób kopię w scratchpadzie, a po nadpisaniu przypomnij o „Revert File”.

## Budowa `Lab4_detekcja.ipynb`

Każdy krok potoku to osobna funkcja i osobna sekcja w układzie: opis → wynik na łatwym RoI → eksperyment → **Wniosek**. Funkcje są w komórce zaczynającej się od `K_GROW = ...`:

| krok | funkcja | co robi |
|---|---|---|
| 1 | `rough_mask` | `S > S_THR` (kanał S z HSV) |
| 2 | `find_candidates` | `findContours`, odrzuca obiekty `< min_area`, wyznacza ROI z marginesem |
| 3–4 | `segment_candidate` | lokalny Otsu ograniczony do konturu wstępnego poszerzonego o `K_GROW`, potem erozja `refine_edges` na kanale B z `delta` = `alpha` × `b_contrast` (mediana B tła ≥ 4 px od maski − średnia B obiektu), na koniec największa składowa nakładająca się na kontur, z wypełnionymi dziurami |
| 5 | `blob_features` + `filter_candidates` | szkielet: najdłuższa ścieżka, nadmiar gałęzi, grubość z transformaty odległości; potem powody odrzucenia |
| całość | `detect_objects` | wywołuje kroki po kolei; domyślne argumenty = **parametry końcowe**; `debug=True` zwraca też `info` z kandydatami, powodami odrzucenia i obrazami pośrednimi |
| 6 (dodatkowy) | `clump_parts`, `split_clump`, `split_clumps` | **poza** `detect_objects`, we własnej komórce w sekcji kroku 6. Dla zlepków odrzuconych w kroku 5: wyższy próg na S w masce zlepka, otwarcie, znaczniki, odbudowa `expand_labels`, najniższy próg z poprawnym podziałem. Zwraca `rescued_*`, `splits_*` |

Krok 6 to decyzja użytkownika: polecenie dotyczy pojedynczych larw, więc kroki 1–5 są wynikiem głównym, a krok 6 jest dodatkową analizą. Analiza instancji liczy się na `objs_calib` / `objs_test`, bez uratowanych larw.

Na co uważać przy zmianach:
- `filter_candidates` nie zmienia wejścia i działa na gotowych cechach. Eksperymenty z progami filtra rób przez nią, bez ponownej segmentacji.
- `blob_features` dokleja margines `FEAT_PAD = 10`, żeby szkielet obiektów przy brzegu liczył się poprawnie.

Kolejność sekcji:
1. obszar roboczy (przycięcie `FLOOR`),
2. „Potok detekcji”: jedna krótka tabela (krok → funkcja → artykuł → parametry końcowe), bez opisów kroków, bo te są w sekcjach kroków. Osobnej sekcji „Odstępstwa od artykułu” już nie ma (usunięta przez użytkownika), odstępstwa są opisane w sekcjach kroków,
3. łatwy RoI,
4. kroki 1 / 2 / 3–4 / 5 z eksperymentami,
5. wyniki dla `calib` i `test` (z analizą strat),
6. wrażliwość wyniku na próg globalny, skrócona na prośbę użytkownika: tabela i wykres dla progów 60–160 oraz krótki wniosek. Szczegółową analizę „skąd nowe larwy przy progu 130” usunięto, bo zastępuje ją krok 6,
7. krok 6 (dodatkowy): rozdzielanie odrzuconych zlepków, ablacja po jednym parametrze, galeria podziałów przyjmowanych tylko bez zabezpieczeń,
8. analiza instancji: `analyze_instance` / `analyze_objects`, walidacja na syntetycznych larwach, eksperyment z erozją a pomiarem.

## Decyzje z uzasadnieniem (nie zmieniać bez nowego eksperymentu)

- **`FLOOR` = (255:2730, 132:3970)**, czyli dno pudełka (275:2710, 152:3950) poszerzone o 20 px z każdej strony.
  - Wycina ścianki i niebieską taśmę, która ma wysokie S.
  - Margines jest po to, żeby nie tracić larw leżących przy ściance (+1 larwa na każdym obrazie, 0 zgubionych, 0 fałszywych obiektów ze ścianek).
  - Dla marginesu 20–60 px wynik jest identyczny. 20 px to najmniejsza wartość, przy której żaden obiekt na `calib` ani `test` nie dotyka brzegu, więc `krawedz` działa już tylko na RoI.
  - Larwa wspinająca się po lewej ściance `test` odpada przez `grubosc` (1.32×), bo maska łapie cień na ściance.
  - Wszystkie współrzędne „w układzie przyciętego obrazu” (RoI, powiększenia, @(x, y) w markdownie) są liczone od tego poszerzonego prostokąta.
- **`S_THR` = Otsu na `calib` = 104.** Dla progów 90–110 wynik jest identyczny poza bladą larwą `calib` @(3546, 1129), która przechodzi filtr przy 90, 104 i 120, a przy pozostałych progach odpada (`grubosc` albo `szkielet`).
- **`min_area` = 1500.** Leży w pustej przerwie rozkładu pól komponentów (ok. 960 → 1730 px).
- **`K_GROW` (ograniczenie lokalnego progu).** Przy 104 nie zmienia wyniku, ale przy wyższym progu globalnym jest konieczne: bez niego przy 130 na `test` przybywają 2 larwy zamiast 6, bo lokalny Otsu (ok. 110) z powrotem skleja larwy rozdzielone w masce wstępnej.
- **Usunięte: dwa otwarcia 3×3** (po progu globalnym i po erozji). Przy progach 70–110 nic nie zmieniały.
- **`refine_alpha` = 0.25, `refine_iters` = 4** (wcześniej stała `refine_delta` = 30; zmiana na prośbę użytkownika, 2026-11-19).
  - Kontrast B tło − larwa w ROI to 62–170, bo pudełko jest nierówno oświetlone. Stała `delta` = 30 to od 0.18 do 0.48 kontrastu, więc na jasnym tle erodowała mocniej.
  - 0.25 daje `delta` ≈ 30 dla mediany kontrastu i średnio tyle samo zdjętych pikseli (5.6% / 5.1%). Krzywa „zdjęto(alpha)” nie ma kolana. Przy 0.15 erozja wcina się w ciało po stronie oświetlonej. Liczba larw dla 0.10–0.50: 84–86 / 89–92.
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
  - Daje 7 nowych larw: 4 dobre, 1 wątpliwą, 2 ze złą maską (larwa przykryta bladą, doklejony fragment bladej sąsiadki).
  - Gubi 1 bladą larwę.
  - Zasada przy budowie bazy instancji: lepiej zgubić larwę niż zapisać złą maskę.
- **Krok 6, parametry** (ablacja w notatniku, wszystkie uratowane maski sprawdzone wzrokowo):
  - progi co 10 (`thr_step`), od pierwszego powyżej progu lokalnego zlepka do 200. Krok 5 daje te same zlepki.
  - `open_k` = 5: przy 0 i 3 ginie para `calib` @(2835, 1810), przy 7 para `test` @(2438, 1700).
  - `grow` = 6: bez odbudowy zostają 2 + 4 larwy, przy 3 px −1 zlepek na obraz, 6 i 10 px dają to samo.
  - `min_part` = 500: zakres 200–1000 bez wpływu.
  - `k_min` = 0.7: odrzuca kawałek larwy przykrytej przez inną (`test` @(393, 1214), 0.49×). Części dobrych podziałów ≥ 0.80×, wynik stały dla 0.6–0.8.
  - `min_cover` = 0.9: odrzuca podziały, w których przy wysokim progu znika cała larwa (`test` @(1028, 273), 0.71) albo jedna przykrywa drugą (`calib` @(3005, 1813), 0.82). Koszt: `calib` @(1032, 172), 2 dobre części, pokrycie 0.50. Dobre podziały ≥ 0.96, wynik stały dla 0.9–0.95.
  - Wersja pierwsza (stały wysoki próg, bez zabezpieczeń) przyjmowała na `test` ok. połowę złych masek (kawałki larw przykrytych przez inne).

## Konwencje pracy (potwierdzone z użytkownikiem)

- **Eksperymenty:** liczone przy parametrach końcowych, każdy zmienia jeden parametr (styl ablacji). Notatnik mówi wprost, że parametry dobierano iteracyjnie. Jeśli efekt kroku znika w dalszej części potoku, pokazuj go na wyniku etapu i na wyniku końcowym.
- **Ablacja w całym zakresie:** sprawdzaj operację w całym zakresie parametrów, nie tylko w punkcie pracy. Ablacja `K_GROW` wyłącznie przy 104 dała błędny wniosek.
- **Biblioteki:** używaj gotowych funkcji OpenCV / skimage / scipy. Własny kod pisz tylko tam, gdzie biblioteki nie mają odpowiednika, np. przy adaptacyjnej erozji z artykułu.
- **Styl markdownu:** pierwsza osoba („dobieram”, „sprawdzam”), zwięźle, wnioski oparte na liczbach.
- **Nazewnictwo:** krok 1 to „binaryzacja wstępna”, a jego wynik to „maska wstępna” / „kontur wstępny” (w kodzie `rough_mask`, `rough`). Nie wracać do „maski zgrubnej”.
- **Liczby w markdownie są wpisane ręcznie.** Po każdej zmianie parametru wykonaj notatnik, porównaj wyjścia tekstowe przed i po, i popraw wszystkie liczby, które się zmieniły. Chodzi o wnioski z kroków 1, 2, 3–4, 5 i 6, tabelę „Wyniki detekcji i dlaczego część larw się zgubiła”, wnioski o wrażliwości na próg, wnioski z eksperymentu z erozją i „Wnioski z analizy”.
- Eksperyment z kroku 1 (próg a maska wstępna) nie ma już `print` (usunął go użytkownik). Liczby z wniosku kroku 1 widać tylko na wykresach, więc sprawdzaj je skryptem.
- **Wizualizacja erozji** (prośba użytkownika): maska po erozji jako półprzezroczysta zieleń, kontur maski z kroku 3 na niebiesko (`find_contours(local, 0.5)`, czyli po granicy pikseli). Pas między nimi to piksele zdjęte. Nie zaznaczać usuniętych pikseli magentą. Funkcja `show_erosion` jest w komórce eksperymentu kroków 3–4. `show_candidate_steps` ma wciąż panel z magentą (nie był ruszany).

## Otwarte wątki

- Krok 6 nie rozróżnia „zniknęła larwa leżąca obok” od „zniknęła larwa, której kawałek dokleił się do sąsiadki”, dlatego `min_cover` gubi 2 dobre larwy w `calib` @(1032, 172).
- Reguła „największa składowa” gubiła larwę `test` @(795, 729). Przy `alpha` = 0.25 cała para @(707, 728) trafia do kroku 6, który odzyskuje obie larwy.
- Znane straty:
  - ciemna larwa w `calib` ok. @(3308, 1716) ma S ≈ 98, czyli tuż pod progiem,
  - jasne larwy tuż po wylince mają niskie S.
- Brak skali na zdjęciach, więc wszystkie pomiary są w pikselach.
