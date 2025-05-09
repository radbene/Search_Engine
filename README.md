# 
Sprawozdanie Techniczne: Silnik Wyszukiwania Dokumentów z Wykorzystaniem SVD
1. Wstęp

Niniejsze sprawozdanie opisuje implementację oraz analizę działania silnika wyszukiwania dokumentów, który wykorzystuje dekompozycję według wartości osobliwych (SVD) w celu poprawy jakości wyszukiwania poprzez redukcję szumu i wymiarowości. System został zaprojektowany do przetwarzania zrzutów Wikipedii i umożliwia porównywanie wyników z metodą standardową (opartą na TF-IDF bez SVD) oraz z SVD dla różnych wartości parametru k.

2. Opis Zastosowanej Metodologii

System składa się z kilku kluczowych modułów: pozyskiwania i przetwarzania danych, budowy macierzy term-dokument, transformacji TF-IDF, normalizacji, opcjonalnej redukcji wymiarowości za pomocą SVD oraz mechanizmu wyszukiwania.

2.1. Pozyskiwanie i Przetwarzanie Wstępne Danych

Źródło danych: System jest przystosowany do pracy ze zrzutami Wikipedii w formacie .xml.bz2. Wykorzystano bibliotekę bz2 do dekompresji archiwów.

Parsowanie XML: Treść artykułów (tytuł i tekst) jest ekstrahowana przy użyciu wyrażeń regularnych. Zaimplementowano funkcję clean_wikitext do usuwania znaczników MediaWiki, linków, szablonów i referencji, co ma na celu uzyskanie czystego tekstu.

Fallback: W przypadku braku zrzutu Wikipedii, system generuje zestaw przykładowych dokumentów, aby umożliwić demonstrację działania.

Tokenizacja i oczyszczanie tekstu: Funkcja preprocess_text odpowiada za:

Konwersję tekstu na małe litery.

Podział tekstu na tokeny (słowa).

Usunięcie angielskich słów stop (ENGLISH_STOP_WORDS z scikit-learn).

Filtrację tokenów krótszych niż 3 znaki oraz tokenów niealfabetycznych.

2.2. Budowa Słownika i Macierzy Term-Dokument

Słownik (Vocabulary): Po przetworzeniu wszystkich dokumentów, budowany jest słownik unikalnych tokenów. Zastosowano filtrowanie słów na podstawie minimalnej (domyślnie 2) i maksymalnej (domyślnie 80% liczby dokumentów) częstotliwości występowania w korpusie. Każdemu słowu w słowniku przypisywany jest unikalny indeks.

Macierz Term-Dokument (TDM): Tworzona jest rzadka macierz (początkowo lil_matrix dla efektywnego budowania, następnie konwertowana do csr_matrix dla efektywnych operacji algebraicznych), gdzie wiersze odpowiadają termom ze słownika, a kolumny dokumentom. Wartości w macierzy reprezentują częstotliwość występowania danego termu w danym dokumencie (TF - Term Frequency).

2.3. Transformacja TF-IDF i Normalizacja

Obliczanie wag IDF: Dla każdego termu w słowniku obliczana jest waga IDF (Inverse Document Frequency) według wzoru: log(N / (df + epsilon)), gdzie N to całkowita liczba dokumentów, df to liczba dokumentów zawierających dany term, a epsilon to mała stała zapobiegająca dzieleniu przez zero.

Macierz TF-IDF: Macierz TDM jest transformowana do macierzy TF-IDF poprzez przemnożenie każdej wartości TF przez odpowiadającą jej wagę IDF.

Normalizacja wektorów: Wektory dokumentów w macierzy TF-IDF (kolumny macierzy) są normalizowane przy użyciu normy L2. Zapewnia to, że długość dokumentu nie wpływa nieproporcjonalnie na miarę podobieństwa.

2.4. Redukcja Wymiarowości za Pomocą SVD

Dekompozycja SVD: Opcjonalnie, na znormalizowanej macierzy TF-IDF (transponowanej, aby dokumenty były w wierszach) stosowana jest dekompozycja SVD. Wykorzystano funkcję svds z scipy.sparse.linalg, która jest przystosowana do macierzy rzadkich.

Wybór komponentów k: Użytkownik może zdefiniować liczbę k największych wartości osobliwych (i odpowiadających im wektorów osobliwych), które zostaną zachowane.

Macierz zredukowana: Wynikiem jest macierz svd_matrix (U * Sigma), gdzie dokumenty są reprezentowane w przestrzeni o niższej wymiarowości (k), oraz macierz svd_model (VT), która służy do transformacji wektorów zapytań do tej samej zredukowanej przestrzeni.

2.5. Przetwarzanie Zapytania i Wyszukiwanie

Wektor zapytania: Wprowadzone przez użytkownika zapytanie jest przetwarzane analogicznie do dokumentów (tokenizacja, usunięcie stopwords, małe litery). Wektor zapytania jest następnie ważony wagami IDF i normalizowany.

Wyszukiwanie podobieństwa:

Bez SVD: Podobieństwo kosinusowe jest obliczane między znormalizowanym wektorem zapytania a wszystkimi znormalizowanymi wektorami dokumentów z macierzy TF-IDF.

Z SVD: Wektor zapytania jest najpierw transformowany do przestrzeni SVD o wymiarowości k (mnożenie przez svd_model.T). Następnie obliczane jest podobieństwo kosinusowe między przetransformowanym wektorem zapytania a wektorami dokumentów z svd_matrix.

Ranking wyników: Dokumenty są sortowane malejąco według obliczonego podobieństwa. Wyniki z podobieństwem poniżej progu 0.1 są odrzucane.

2.6. Interfejs Użytkownika

Aplikacja posiada graficzny interfejs użytkownika zaimplementowany w bibliotece Tkinter. Umożliwia on:

Wskazanie ścieżki do zrzutu Wikipedii.

Określenie maksymalnej liczby dokumentów do przetworzenia.

Budowanie, zapisywanie i ładowanie indeksu.

Wprowadzanie zapytań.

Wybór liczby wyników do wyświetlenia.

Włączenie/wyłączenie SVD oraz wybór liczby komponentów k za pomocą suwaka.

Wyświetlanie wyników wyszukiwania wraz z tytułem, indeksem dokumentu, wynikiem podobieństwa oraz podglądem treści.

### 3. Konfiguracja Eksperymentów i Prezentacja Wyników

Do oceny systemu przeprowadzono serię testów na zbiorze danych pochodzącym ze zrzutu _simplewiki_ (lub danych przykładowych, jeśli zrzut nie był dostępny). Użyto trzech różnych zapytań:
1.  "bloody battle in middle ages"
2.  "best footballer of all time"
3.  "Donald Trump"

Porównano wyniki uzyskane metodą standardową (bez SVD) z wynikami przy użyciu SVD dla `k` równego 10, 50, 100, 150 oraz 200. Wyniki (top 10 dokumentów wraz z ich tytułami i wartościami podobieństwa) przedstawiono w poniższych tabelach.

#### 3.1. Wyniki dla zapytania: "bloody battle in middle ages"

| Pozycja | Bez SVD                                         | SVD k = 10                               | SVD k = 50                               | SVD k = 100                                  | SVD k = 150                                  | SVD k = 200                                     |
|:-------:|:------------------------------------------------|:-----------------------------------------|:-----------------------------------------|:---------------------------------------------|:---------------------------------------------|:------------------------------------------------|
| 1       | My bloody valentine (0.4826)                    | Jayadeva Malla (0.5009)                  | French and Indian War (0.5633)           | World War I (0.4768)                         | Victoria Cross for Australia (0.5411)        | Victoria Cross for Australia (0.5352)           |
| 2       | Bloody Sunday (0.4665)                          | Andra Martin (0.4837)                    | August 26 (0.5594)                       | World War II (0.4726)                        | Russo-Turkish war (0.5115)                   | Template:USCabinet (0.4872)                     |
| 3       | High Middle Ages (0.4501)                       | Sergio Fantoni (0.4559)                  | August 23 (0.5547)                       | 1943 (0.4717)                                | Template:USCabinet (0.5049)                  | Russo-Turkish war (0.4867)                      |
| 4       | Bloody Mary (cocktail) (0.4239)                 | Mizuo Peck (0.4524)                      | January 26 (0.5488)                      | 1813 (0.4350)                                | World War II (0.4992)                        | Battle of Yorktown (0.4601)                     |
| 5       | Category:Battles of the Middle Ages (0.4175)    | Chloe Bennet (0.4470)                    | December 18 (0.5445)                     | Wehrmacht (0.4201)                           | World War I (0.4898)                         | 1260s (0.4559)                                  |
| 6       | Template:Tank battles (0.3775)                  | Sumi Haru (0.4442)                       | Jacques (0.5386)                         | Luftwaffe (0.4194)                           | List of infantry guns (0.4783)               | Libertadores (0.4488)                           |
| 7       | Archduke Charles, Duke of Teschen (0.3649)      | Northern Calloway (0.4430)               | June 14 (0.5350)                         | European theatre of World War I (0.4135)     | Libertadores (0.4724)                        | List of infantry guns (0.4443)                  |
| 8       | Bloody Mary (0.3529)                            | Thomas Gibson (0.4418)                   | March 21 (0.5332)                        | Nazi Germany (0.4003)                        | Template:War crimes (0.4697)                 | Template:War crimes (0.4384)                    |
| 9       | 6th Army (Wehrmacht) (0.3234)                   | Franco Graziosi (0.4408)                 | June 27 (0.5326)                         | September 13 (0.3926)                        | List of field guns (0.4581)                  | Napoleonic Wars (0.4345)                        |
| 10      | Cristero War (0.3206)                           | Laurie Mitchell (0.4390)                 | March 29 (0.5317)                        | August 27 (0.3836)                           | Interwar period (0.4490)                     | Template:British colonial campaigns (0.4307)    |

#### 3.2. Wyniki dla zapytania: "best footballer of all time"

| Pozycja | Bez SVD                                            | SVD k = 10                               | SVD k = 50                               | SVD k = 100                                  | SVD k = 150                                     | SVD k = 200                                     |
|:-------:|:---------------------------------------------------|:-----------------------------------------|:-----------------------------------------|:---------------------------------------------|:------------------------------------------------|:------------------------------------------------|
| 1       | 2021 in association football deaths (0.7606)       | Andra Martin (0.5239)                    | June 27 (0.5052)                         | Module:Football box/styles.css (0.8497)      | Template:Time ago/core (0.8572)                 | Module:Football box/styles.css (0.8272)         |
| 2       | 2024 in association football deaths (0.7456)       | Jayadeva Malla (0.5000)                  | January 26 (0.4994)                      | Template:Time ago/core (0.8497)              | Module:Football box/styles.css (0.8572)         | Template:Time ago/core (0.8272)                 |
| 3       | Yamaguchi (name) (0.7172)                          | Matthew Faber (0.4926)                   | June 1 (0.4962)                          | Template:Current time (0.8490)               | Template:Current time (0.8567)                  | Template:Current time (0.8266)                  |
| 4       | Sudo (surname) (0.7065)                            | Claude Evrard (0.4840)                   | June 16 (0.4961)                         | Category:Deaths by time (0.8139)             | Category:Deaths by time (0.8125)                | Ecclesiastes (0.7843)                           |
| 5       | Gary Ablett (disambiguation) (0.5726)              | Sumi Haru (0.4814)                       | August 23 (0.4891)                       | Ecclesiastes (0.8060)                        | Ecclesiastes (0.8117)                           | Category:Deaths by time (0.7821)                |
| 6       | Ryohei Suzuki (0.5146)                             | Sergio Fantoni (0.4805)                  | December 18 (0.4874)                     | Category:Births by time (0.7931)             | Template:User time zone (0.7999)                | Template:User time zone (0.7691)                |
| 7       | Hitoshi Sasaki (0.4772)                            | Chloe Bennet (0.4794)                    | July 1 (0.4873)                          | Template:User time zone (0.7926)             | Category:Births by time (0.7896)                | Category:Births by time (0.7597)                |
| 8       | Kubo (surname) (0.4620)                            | Shwikar (0.4792)                         | March 21 (0.4858)                        | Template:Time zones of Europe (0.7814)       | Template:Daylight saving active/doc (0.7838)    | Template:Time zones of Europe (0.7596)          |
| 9       | Akira Matsunaga (0.4612)                           | Franco Graziosi (0.4784)                 | July 13 (0.4858)                         | Template:Daylight saving active/doc (0.7767) | Template:Time zones of Europe (0.7810)          | Template:Daylight saving active/doc (0.7529)    |
| 10      | Kawai (name) (0.4512)                              | Wladimir Yordanoff (0.4752)              | October 9 (0.4838)                       | Template:Time measurement and standards (0.7714) | Template:Time measurement and standards (0.7799) | Template:Time measurement and standards (0.7518)|

#### 3.3. Wyniki dla zapytania: "Donald Trump"

| Pozycja | Bez SVD                                          | SVD k = 10                               | SVD k = 50                                        | SVD k = 100                                     | SVD k = 150                                     | SVD k = 200                                          |
|:-------:|:-------------------------------------------------|:-----------------------------------------|:--------------------------------------------------|:------------------------------------------------|:------------------------------------------------|:-----------------------------------------------------|
| 1       | Family of Donald Trump (0.8042)                  | Andra Martin (0.5971)                    | Category:Scottish National Party people (0.6371)  | Category:Festivals in the United States (0.5976)| Category:1933 in the United States (0.4411)     | List of vice presidents of the United States (0.6054)|
| 2       | Donald Trump (disambiguation) (0.7447)           | Jayadeva Malla (0.5647)                  | List of political parties in Australia (0.6083)   | Category:1982 in the United States (0.5976)     | Category:1930 in the United States (0.4411)     | Lieutenant Governor of South Dakota (0.6035)         |
| 3       | Trump (0.7367)                                   | Sergio Fantoni (0.5522)                  | List of political parties in the United States (0.6026)| Category:Disestablishments in the United States (0.5976)| Category:1920 in the United States (0.4411)     | President-elect of the United States (0.5947)        |
| 4       | Never Trump movement (0.7351)                    | Chloe Bennet (0.5478)                    | Know-Nothing Party (0.5966)                       | Category:1956 in the United States (0.5976)     | Category:1921 in the United States (0.4411)     | List of lieutenant governors of Ohio (0.5945)        |
| 5       | Impeachment of Donald Trump (0.7152)             | Sumi Haru (0.5474)                       | Lieutenant Governor of New York (0.5895)          | Category:1954 in the United States (0.5976)     | Category:1966 in the United States (0.4411)     | Template:Party shading/Silver (0.5919)               |
| 6       | Trump Tower (0.6865)                             | Matthew Faber (0.5470)                   | Electional history of Kurdish parties in Turkey (0.5873)| Category:1966 in the United States (0.5914)     | Category:1957 in the United States (0.4411)     | List of lieutenant governors of Vermont (0.5918)     |
| 7       | Donald Trump (0.6452)                            | Scoey Mitchell (0.5421)                  | 2016 United States presidential election (0.5825) | Category:1920 in the United States (0.5914)     | Category:1956 in the United States (0.4364)     | List of United States representatives from Kansas (0.5906)|
| 8       | The Trump Organization (0.6402)                  | Franco Graziosi (0.5416)                 | List of political parties in the Bahamas (0.5806) | Category:1933 in the United States (0.5914)     | Category:Festivals in the United States (0.4364)| 2016 Republican National Convention (0.5859)         |
| 9       | Attempted assassination of Donald Trump (0.6270) | Max Grodénchik (0.5377)                  | Liberal Party (0.5781)                            | Category:1957 in the United States (0.5914)     | Category:1982 in the United States (0.4364)     | List of presidents of the United States (0.5845)     |
| 10      | 2016 United States presidential election in Florida (0.6085) | Marc Menchaca (0.5363) | Leader of the Opposition (Japan) (0.5771)         | Category:1921 in the United States (0.5914)     | Category:Disestablishments in the United States (0.4364)| Lieutenant Governor of North Dakota (0.5825)         |
4. Analiza Wyników i Wnioski

Analiza przedstawionych wyników pozwala sformułować następujące wnioski:

Wpływ wartości k na jakość wyników:

Niskie k (np. 10, 50): Dla wszystkich zapytań, SVD z niskimi wartościami k generalnie prowadzi do znaczącego pogorszenia jakości wyników. Wyniki często wydają się losowe lub związane jedynie z bardzo ogólnymi koncepcjami (np. daty, ogólne kategorie, nazwiska, które mogły pojawić się w kontekście wielu tematów). Sugeruje to, że zbyt agresywna redukcja wymiarowości prowadzi do utraty kluczowych informacji semantycznych potrzebnych do rozróżnienia trafnych dokumentów.

Średnie k (np. 100): Dla zapytania "bloody battle in middle ages", wyniki nadal są słabe, koncentrując się na wojnach światowych i datach. Dla "best footballer of all time", wyniki są zdominowane przez szablony i kategorie związane z "czasem" oraz "football", ale bez konkretnych piłkarzy. Dla "Donald Trump", wyniki są bardzo ogólne (kategorie dotyczące USA w różnych latach/aspektach). Wydaje się, że przy k=100 system nadal ma trudności z uchwyceniem specyfiki zapytań.

Wysokie k (np. 150, 200):

Dla "bloody battle in middle ages", wyniki dla k=150 i k=200 zaczynają zawierać więcej artykułów o tematyce militarnej (np. "Victoria Cross for Australia", "Russo-Turkish war", "Battle of Yorktown"), ale nadal nie są one ściśle związane ze średniowieczem.

Dla "best footballer of all time", wyniki pozostają zdominowane przez tematykę "czasu" i ogólne szablony piłkarskie, co sugeruje, że słowo "time" w zapytaniu silnie wpływa na wyniki SVD, przyćmiewając "footballer".

Dla "Donald Trump", przy k=200 pojawiają się bardziej relewantne wyniki związane z polityką USA (np. "List of vice presidents", "President-elect", "2016 Republican National Convention"), choć wyniki dla k=150 są ponownie bardzo ogólne.

Porównanie SVD z metodą standardową (bez SVD):

"bloody battle in middle ages": Metoda standardowa, mimo że nieidealna, dostarcza bardziej trafne wyniki (np. "High Middle Ages", "Category:Battles of the Middle Ages") niż SVD dla testowanych wartości k. SVD wydaje się gubić kontekst "middle ages".

"best footballer of all time": Metoda standardowa zdecydowanie przewyższa SVD. Znajduje listy zmarłych piłkarzy, nazwiska i strony ujednoznaczniające związane z piłką nożną. SVD, niezależnie od k (powyżej 50), jest zdominowane przez słowo "time".

"Donald Trump": Metoda standardowa dostarcza bardzo trafne i bezpośrednio powiązane wyniki (rodzina Trumpa, strony ujednoznaczniające, ruchy polityczne, wydarzenia). SVD, nawet dla k=200, dostarcza wyniki bardziej ogólne, związane z systemem politycznym USA, ale mniej bezpośrednio z samą osobą. Dla niższych k, wyniki są znacznie gorsze.

Ogólne obserwacje dotyczące SVD:

Czułość na słowa kluczowe: Wydaje się, że SVD może nadmiernie wzmacniać wpływ pewnych ogólnych słów (np. "time" w zapytaniu o piłkarza), jeśli są one silnie reprezentowane w głównych komponentach osobliwych, co prowadzi do dryfowania tematycznego.

Utrata specyfiki: SVD, dążąc do uchwycenia ukrytych struktur semantycznych i redukcji szumu, może jednocześnie tracić specyficzne informacje, które są kluczowe dla trafności niektórych zapytań.

Brak uniwersalnie optymalnego k: Nie zaobserwowano jednej wartości k, która działałaby dobrze dla wszystkich typów zapytań. Optymalne k prawdopodobnie zależy od charakterystyki korpusu i samego zapytania.

Baseline TF-IDF: W przypadku testowanych zapytań, prosta metoda TF-IDF bez SVD okazała się być konkurencyjna, a często nawet lepsza, szczególnie gdy zapytanie zawierało unikalne lub specyficzne termy.

5. Podsumowanie

Zaimplementowany silnik wyszukiwania pozwala na eksplorację wpływu SVD na jakość wyników. Przeprowadzone testy wskazują, że dla badanych zapytań i wybranego zakresu wartości k, zastosowanie SVD nie przyniosło jednoznacznej poprawy w stosunku do standardowej metody TF-IDF. W wielu przypadkach, szczególnie dla niskich i średnich wartości k, wyniki uległy pogorszeniu. Dla zapytań wymagających odnalezienia konkretnych encji lub gdy zapytanie zawierało terminy o dużej sile dyskryminacyjnej, metoda TF-IDF bez redukcji wymiarowości okazała się skuteczniejsza. SVD może być bardziej podatne na wpływ ogólnych, często występujących słów w zapytaniu, co może prowadzić do mniej precyzyjnych wyników.