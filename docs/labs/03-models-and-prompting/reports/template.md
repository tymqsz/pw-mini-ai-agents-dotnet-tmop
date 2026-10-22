# Raport — Laboratorium 03

**Zespół:** Tymon Piwowarski + Michał Obrębski
**Data:** 22.10.2026  

## Środowisko

| Parametr | Wartość |
|----------|---------|
| Wersja Ollama | 0.40.8 |
| Sprzęt (CPU, RAM) | AMD Ryzen 5 3600 6-Core, 16GB RAM |
| Liczba dostępnych notatek | 170 |

## EXP-00 — Sposób oceny wyników

Jak oceniamy poprawność odpowiedzi modelu? Jakie metryki przyjmujemy i dlaczego?

Poprawność odpowiedzi modelu oceniamy poprzez automatyczne porównanie wyekstrahowanych pól ze zbiorem wzorcowym (*ground truth*) dla każdej notatki. Ponieważ zadanie polega na ekstrakcji ustrukturyzowanej informacji z tekstu do formatu JSON, oceniamy zarówno poprawność składniową odpowiedzi, trafność poszczególnych pól, jak i determinizm działania modelu oraz czas odpowiedzi.

| Metryka | Definicja |
|---------|-----------|
| **Poprawny JSON** (*Valid JSON*) | Odsetek (lub liczba) odpowiedzi modelu, które udało się poprawnie zdeserializować jako prawidłowy obiekt JSON zgodny z oczekiwaną strukturą. |
| **Floor accuracy** | Odsetek notatek, dla których model zwrócił dokładną wartość oczekiwaną dla pola `floor` (liczba całkowita oznaczająca piętro lub `null` w przypadku braku jednoznacznej informacji). |
| **Elevator accuracy** | Odsetek notatek, dla których model zwrócił dokładną wartość oczekiwaną dla pola `elevator` (`true`, `false` lub `null`, gdy notatka nie wspomina o windzie). |
| **Exact match** | Odsetek notatek, dla których model poprawnie rozpoznał jednocześnie oba pola: `floor` oraz `elevator`. Jest to kluczowa metryka jakościowa, gdyż tylko w pełni poprawny rekord nie wymaga korekty. |
| **Powtarzalne notatki** | Liczba notatek (na 10 badanych), dla których model w 5 niezależnych uruchomieniach zwrócił za każdym razem identyczną odpowiedź (miara stabilności i determinizmu). |
| **Średni czas [ms]** | Średni czas generowania odpowiedzi dla pojedynczej notatki wyrażony w milisekundach (miara wydajności i opóźnienia inferencji). |

## EXP-01 — Porównanie modeli

| Model | Floor accuracy | Elevator accuracy | Exact match | Średni czas [ms] |
|-------|----------------|-------------------|-------------|------------------|
| `nemotron-3-nano:4b` | 100% | 90% | 90% | 3680 |
| `lfm2.5-thinking` | 30% | 20% | 10% | 9510 |

**Exact match** oznacza odsetek notatek, dla których model poprawnie rozpoznał jednocześnie oba pola: `floor` i `elevator`.

**Wnioski:**
* Model **`nemotron-3-nano:4b`** zdecydowanie wygrywa pod każdym względem: osiąga **90% Exact match** (100% trafności dla piętra i 90% dla windy) oraz w 10/10 przypadków zwraca poprawny format JSON.
* Model **`lfm2.5-thinking`** okazał się nieprzydatny w wariancie `zero-shot` do bezpośredniej ekstrakcji JSON — uzyskał zaledwie **10% Exact match**, a tylko połowa jego odpowiedzi (5/10) stanowiła poprawny JSON (modele typu *thinking* często dodają znaczniki rozumowania lub tekst poboczny, co psuje parsowanie).
* Pod względem wydajności `nemotron-3-nano:4b` jest **ponad 2,5-krotnie szybszy** (3680 ms vs 9510 ms na zapytanie), co wynika z braku narzutu na generowanie długiego łańcucha myśli (*chain-of-thought*).

## EXP-02 — Prompting

| Wariant | Floor accuracy | Elevator accuracy | Exact match |
|---------|----------------|-------------------|-------------|
| zero-shot | 100% | 90% | 90% |
| one-shot | 100% | 90% | 90% |
| few-shot | 100% | 100% | 100% |

**Wnioski:**
* W wariantach **zero-shot** oraz **one-shot** model `nemotron-3-nano:4b` bezbłędnie poradził sobie z 9 na 10 notatek, w tym z trudniejszymi przypadkami takimi jak rzymska numeracja (`IV piętro`), piętro podziemne (`-1`), niejednoznaczność (`Drugie lub trzecie piętro` jako `null`) czy ograniczony zasięg windy (`4 piętro, winda tylko do 3`).
* Jedynym błędem w `zero-shot` i `one-shot` była notatka **NOTE-010** (`"Winda nie działa, dostawa na 5p"`), gdzie model interpretował niedziałającą windę jako jej brak (`elevator=false`), podczas gdy oczekiwana wartość to `elevator=true` (winda fizycznie znajduje się w budynku).
* Dodanie kilku przykładów w wariancie **few-shot** pozwoliło doprecyzować tę regułę domenową i podniosło wynik **Exact match do 100%** przy zachowaniu zbliżonego czasu odpowiedzi (3519 ms).

## EXP-03 — Temperature

| Temperature | Exact match | Powtarzalne notatki |
|-------------|-------------|---------------------|
| 0.0 | 90% | 10/10 |
| 0.5 | 82% | 8/10 |
| 1.0 | 78% | 6/10 |

**Wnioski:**
* Parametr **Temperature = 0.0** zapewnia **100% powtarzalności (10/10)** — dla każdej notatki w 5 próbach model zwracał dokładnie ten sam wynik, osiągając najwyższy wynik **Exact match (90%)**.
* Wraz ze wzrostem temperatury spada zarówno stabilność odpowiedzi, jak i ich dokładność: przy **Temperature = 0.5** powtarzalność spada do 8/10 (Exact match: 82%), a przy **Temperature = 1.0** do 6/10 (Exact match: 78%).
* Wyższa temperatura sprzyja halucynacjom na prostych przypadkach — np. dla `NOTE-006` (`"Mieszkanie 15, domofon 15"`) przy `T=0.5` model w 3 na 5 prób błędnie przypisał numer mieszkania jako piętro (`floor=15`), a przy `T=1.0` zrobił ten błąd we wszystkich 5 próbach. W zadaniach ekstrakcji danych temperatura powinna być zawsze ustawiona na `0.0`.

## EXP-04 — Structured output — JSON i walidacja

| Wariant | Poprawny JSON | Exact match |
|---------|---------------|-------------|
| prompt | 10/10 (100%) | 90% |
| json-schema | 10/10 (100%) | 90% |

**Wnioski:**
* W obu wariantach (`prompt` oraz `json-schema`) model `nemotron-3-nano:4b` zwrócił w **100% poprawny składniowo JSON (10/10)** oraz uzyskał identyczną trafność merytoryczną (**Exact match: 90%**).
* Zastosowanie wymuszenia struktury na poziomie silnika inferencji (**`json-schema`**) przyniosło jednak wyraźną korzyść wydajnościową — średni czas odpowiedzi spadł z **3667 ms do 2971 ms** (przyspieszenie o blisko 19%), ponieważ model ma ograniczoną przestrzeń dozwolonych tokenów podczas generowania.
* Ponadto w systemie produkcyjnym `json-schema` daje twardą gwarancję zgodności typów danych i chroni przed sytuacją, w której model dopisze komentarz poza strukturą JSON.

## Najtrudniejsze notatki

Które notatki model najczęściej interpretował błędnie i dlaczego?

1. **NOTE-010 (`"Winda nie działa, dostawa na 5p"`) — najczęstszy błąd merytoryczny:**
   * **Oczekiwano:** `floor=5, elevator=true`
   * **Odpowiedź modelu (zero-shot / one-shot):** `floor=5, elevator=false`
   * **Dlaczego:** Notatka zawiera pułapkę semantyczną. Z punktu widzenia kuriera "winda nie działa" oznacza konieczność wejścia po schodach, dlatego model bez dodatkowych instrukcji logicznie przyjmuje `elevator=false`. Z punktu widzenia bazy danych o budynku informacja ta dowodzi jednak, że winda w budynku fizycznie istnieje (`elevator=true`), a jedynie uległa awarii. Dopiero pokazanie wzorca w `few-shot` wyeliminowało ten błąd.
2. **NOTE-006 (`"Mieszkanie 15, domofon 15"`) — podatność na halucynacje przy wyższej temperaturze:**
   * **Oczekiwano:** `floor=null, elevator=null`
   * **Odpowiedź modelu (T=0.5 oraz T=1.0):** `floor=15, elevator=null`
   * **Dlaczego:** W notatce pojawia się liczba `15`, ale dotyczy numeru lokalu i domofonu, a nie kondygnacji. Przy zwiększonej losowości (`temperature` powyżej 0) model zbyt agresywnie próbuje dopasować jedyną dostępną w tekście liczbę do pola `floor`.
3. **NOTE-003 (`"Parter, wejście od podwórza"`), NOTE-001 (`"Trzecie piętro..."`) oraz NOTE-005 (`"Winda na końcu korytarza"`):**
   * Przy `temperature=0.5` oraz `1.0` pojawiały się pojedyncze niespójności w mapowaniu słownym (np. `Parter` zmapowany raz jako `1` lub `null` zamiast `0`, `"Trzecie piętro"` jako `2`, czy losowe wpisanie `floor=5` przy braku informacji o piętrze w `NOTE-005`). Przy `temperature=0.0` problemy te w ogóle nie występowały.

## Rekomendacja

Który model i prompt wybieramy do projektu — i jakie mamy dowody, że to dobry wybór?

Do wdrożenia w projekcie wybieramy następującą konfigurację:
* **Model:** `nemotron-3-nano:4b`
* **Strategia promptowania:** `few-shot`
* **Parametr Temperature:** `0.0`
* **Format wyjściowy:** `json-schema`

**Dowody potwierdzające słuszność wyboru:**
1. **Najwyższa skuteczność merytoryczna:** Model `nemotron-3-nano:4b` już w wariancie bazowym osiąga 90% Exact match (wobec 10% dla `lfm2.5-thinking`), a po zastosowaniu promptu **few-shot** osiąga **100% Exact match**, poprawnie obsługując nawet niejednoznaczne przypadki brzegowe (jak awaria windy w `NOTE-010`).
2. **Pełny determinizm i brak halucynacji:** Eksperyment EXP-03 dowiódł, że ustawienie **`temperature=0.0`** gwarantuje 10/10 powtarzalnych odpowiedzi i całkowicie eliminuje problem mylenia numeru mieszkania z piętrem (`NOTE-006`).
3. **Szybkość i stabilność integracji:** Eksperyment EXP-04 wykazał, że użycie **`json-schema`** zapewnia 100% poprawności struktury wyjściowej, jednocześnie skracając średni czas generowania odpowiedzi o ok. 700 ms (z 3667 ms do 2971 ms).