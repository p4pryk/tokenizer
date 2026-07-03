# Raport z Prac i Specyfikacja: Byte-Level BPE Tokenizer (Polish)

Niniejszy dokument stanowi podsumowanie i opis inżynieryjny prac nad autorskim, zaawansowanym tokenizerem **Byte-level BPE (BBPE)** napisanym od podstaw w czystym Pythonie na potrzeby projektu małego polskiego modelu językowego (LLM ok. 200M parametrów).

---

## 1. Wybór architektury: BPE na poziomie znaków vs Byte-level BPE

Podczas prac zaimplementowano dwa rodzaje tokenizerów, ostatecznie wybierając standard komercyjny (**BBPE**):

*   **Początkowy (Character-Level BPE):**
    Rozpoczynał naukę bezpośrednio od znaków tekstowych Unicode. Choć prosty dydaktycznie, posiada krytyczną wadę: gdy użytkownik wprowadzi znak niewidziany podczas treningu (rzadkie emoji, obcy alfabet, rzadki symbol matematyczny), model zwraca wadliwy token `<UNK>` (ID 0).
*   **Docelowy (Byte-Level BPE):**
    Tekst jest najpierw kodowany na ciąg bajtów UTF-8. Bazowy alfabet to zawsze **stałe 256 bajtów (0-255)**. 
    *   **Zaleta:** Dowolny znak (i plik) na świecie można wyrazić za pomocą bajtów. Dzięki temu tokenizer **nigdy nie rzuca błędu `<UNK>`** i bezbłędnie koduje każdy wejściowy tekst.
    *   **Widoczność w JSON (bytes_to_unicode):** Zaimplementowano mapowanie bajtów na czytelne dla oka znaki Unicode (stosowane np. w GPT-2), dzięki czemu znaki kontrolne i spacje są prezentowane w pliku JSON w cywilizowany sposób (np. spacja jako symbol `Ġ`).

---

## 2. Podział Zapasów Treningowych (Dataset Balancing)

Trening przeprowadzono na potężnym polskim korpusie **Polish DynaWord** (łącznie ok. 6 GB zróżnicowanego tekstu w 12 domenach, m.in. akty prawne EUR-lex, Dziennik Ustaw, Wikisłownik, polska literatura Wolne Lektury, Wikipedia, Wikinews).

*   **Selektywne Próbkowanie (Balancing):**
    Aby uniknąć zdominowania słownika przez żargon prawniczy (EUR-Lex i Parlament stanowią ponad 60% całego korpusu) oraz chronić pamięć RAM przed zapchaniem, wdrożono funkcję `load_balanced_polish_corpus` pobierającą równe porcje po **500 000 znaków** z każdego pliku Parquet.
*   **Całkowity zbiór treningowy:** Ponad **6 000 000 znaków** (dokładnie 6 067 340 znaków).

---

## 3. Zabezpieczenia przed Overfittingiem Słownika (Wyrzucenie spamu EUR-Lex)

Podczas zwiększania rozmiaru słownika, standardowy algorytm BPE zaczął scalić powtarzające się formułki urzędowe w gigantyczne, kilkudziesięcio-znakowe tokeny (np. *„oraz do zarządzenia jej publikacji w Dzienniku Urzędowym...”* z podpisem przewodniczącego EUR-Lex). 
Aby zapobiec marnowaniu slotów w słowniku, zaimplementowano dwa rewolucyjne ograniczenia w pętli treningowej:
1.  **Długość maksymalna tokenu (`max_token_len` = 20):** Algorytm ignoruje i odrzuca scalenia dłuższe niż 20 znaków rzeczywistego tekstu. Pozwala to na naukę słów i ich odmian, odrzucając całe zdania.
2.  **Zakaz nowej linii (`\n`):** Zabroniono łączenia linii i akapitów. Tokeny kończą się przed przejściem do nowej linii.

---

## 4. Wyniki Ewaluacji (Test na całym "Panu Tadeuszu")

Wyniki testów na pełnym eposie narodowym (447k znaków, 69k słów) po wytrenowaniu ostatecznego słownika **`vocab_size = 7500`** na pełnej, zbalansowanej porcji DynaWord (6 mln znaków):

*   **Wierność rekonstrukcji (Wycofanie / Dekodowanie):** **100.0% (IDEALNA)**.
*   **Liczba nieznanych tokenów (`<UNK>`):** **0** (Idealne pokrycie bajtowe, całkowite wyeliminowanie problemu nieznanych znaków).
*   **Współczynnik kompresji znakowej:** **63.66%** (tekst w postaci tokenów zajmuje ponad 2.75-krotnie mniej miejsca!).
*   **Wskaźnik Fertility:** **2.353 tok/słowo** (Znakomita kompresja — słownik potrzebuje średnio zaledwie 2.35 tokenu do zakodowania całego bogatego polskiego słowa!).

### Próbka podziału tekstu (Inwokacja):
*   `Adam Mickiewicz` $\rightarrow$ `['A', 'dam ', 'Mi', 'ckie', 'wicz']`
*   `Pan Tadeusz` $\rightarrow$ `['Pan', ' T', 'ade', 'usz']` (*Słowo "Pan" skompresowane w całości!*)
*   `czyli ostatni zajazd na Litwie` $\rightarrow$ `['czyli ', 'ostat', 'ni ', 'za', 'jazd ', 'na ', 'L', 'it', 'wie']` (*Spójniki i przyimki kodowane jako całe słowa ze spacjami!*)

---