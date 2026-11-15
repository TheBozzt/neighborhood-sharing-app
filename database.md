# Dokumentacja Projektowa - Baza danych
**Projekt:** Aplikacja do sąsiedzkiego współdzielenia sprzętu

---

## 1. Pojęciowy Model Danych

```mermaid
erDiagram
    OSIEDLE ||--o{ UZYTKOWNIK : "ma członków"
    OSIEDLE ||--o{ KOD_ZAPROSZENIA : "wydaje"
    OSIEDLE ||--o{ PRZEDMIOT : "posiada w katalogu"
    OSIEDLE ||--o{ OGLOSZENIE : "publikuje na tablicy"
    UZYTKOWNIK ||--o{ PRZEDMIOT : "udostępnia"
    UZYTKOWNIK ||--o{ WYPOZYCZENIE : "uczestniczy (właściciel/pożyczający)"
    UZYTKOWNIK ||--o{ URZADZENIE : "otrzymuje powiadomienia na"
    UZYTKOWNIK ||--o{ WIADOMOSC : "wysyła"
    UZYTKOWNIK ||--o{ OCENA : "wystawia i otrzymuje"
    UZYTKOWNIK ||--o{ ZGLOSZENIE : "zgłasza do moderacji"
    PRZEDMIOT ||--o{ WYPOZYCZENIE : "posiada historię rezerwacji"
    WYPOZYCZENIE ||--o{ KOD_POTWIERDZENIA : "wymaga weryfikacji przez"
    WYPOZYCZENIE ||--o{ WIADOMOSC : "posiada dedykowany czat"
    WYPOZYCZENIE ||--o{ OCENA : "kończy się wystawieniem"
    WYPOZYCZENIE |o--o{ ZGLOSZENIE : "może być powodem"

    OSIEDLE {
        Tekst nazwa
        Tekst adres
        Tekst regulamin
        Data data_utworzenia
    }
    KOD_ZAPROSZENIA {
        Tekst hash_tokenu
        Tekst typ_kodu
        Data data_waznosci
        Zmienna_logiczna status_uniewaznienia
    }
    UZYTKOWNIK {
        Tekst imie
        Tekst nazwisko
        Tekst numer_mieszkania
        Tekst adres_email
        Tekst hash_hasla
        Zmienna_logiczna status_weryfikacji_email
        Tekst rola_systemowa
        Tekst status_konta
        Data data_zanonimizowania
    }
    URZADZENIE {
        Tekst token_fcm
    }
    PRZEDMIOT {
        Tekst nazwa
        Tekst opis_stanu
        Tekst kategoria
        Tekst status_widocznosci
        Tekst status_archiwalny
        Lista_Zdjec zdjecia
    }
    OGLOSZENIE {
        Tekst tresc
        Data data_utworzenia
    }
    WYPOZYCZENIE {
        Okres_czasu okres_zajetosci_kalendarza
        Tekst status_rezerwacji
        Zmienna_logiczna ocena_wystawiona_przez_wlasciciela
        Zmienna_logiczna ocena_wystawiona_przez_pozyczajacego
        Lista_Zdjec zdjecia_z_wydania
        Lista_Zdjec zdjecia_ze_zwrotu
    }
    KOD_POTWIERDZENIA {
        Tekst etap_wymiany
        Tekst hash_tokenu
        Data data_waznosci
    }
    WIADOMOSC {
        Tekst tresc
        Plik zalacznik_zdjeciowy
        Data data_wyslania
    }
    ZGLOSZENIE {
        Tekst powod_zgloszenia
        Tekst opis_problemu
        Tekst wyjasnienie_drugiej_strony
        Tekst decyzja_moderatora
        Tekst uzasadnienie_decyzji
        Lista_Zdjec zdjecia_dowodowe
    }
    OCENA {
        Liczba liczba_gwiazdek
        Tekst komentarz
        Zmienna_logiczna status_ukrycia
    }
```

---

## 2. Logiczny Model Danych
Model logiczny zgodny i spójny z modelem pojęciowym. Zawiera definicje kluczy głównych, kluczy obcych oraz typy danych.

```mermaid
erDiagram
    OSIEDLE ||--o{ UZYTKOWNIK : "ma członków"
    OSIEDLE ||--o{ KOD_ZAPROSZENIA : "wydaje"
    OSIEDLE ||--o{ PRZEDMIOT : "katalog"
    OSIEDLE ||--o{ OGLOSZENIE : "tablica"
    UZYTKOWNIK ||--o{ PRZEDMIOT : "posiada"
    UZYTKOWNIK ||--o{ WYPOZYCZENIE : "pożycza"
    UZYTKOWNIK ||--o{ URZADZENIE : "tokeny push"
    UZYTKOWNIK ||--o{ WIADOMOSC : "wysyła"
    UZYTKOWNIK ||--o{ OCENA : "otrzymuje/wystawia"
    UZYTKOWNIK ||--o{ ZGLOSZENIE : "tworzy spór"
    PRZEDMIOT ||--o{ WYPOZYCZENIE : "historia"
    WYPOZYCZENIE ||--o{ KOD_POTWIERDZENIA : "QR przy wymianie"
    WYPOZYCZENIE ||--o{ WIADOMOSC : "czat"
    WYPOZYCZENIE ||--o{ OCENA : "maksymalnie dwie"
    WYPOZYCZENIE |o--o{ ZGLOSZENIE : "spory"

    OSIEDLE {
        bigint id PK
        varchar nazwa
        varchar adres
        text regulamin
        timestamptz utworzone
    }
    KOD_ZAPROSZENIA {
        bigint id PK
        bigint osiedle_id FK
        char token_hash UK
        varchar typ "QR, LINK"
        timestamptz wazny_do
        boolean uniewazniony
    }
    UZYTKOWNIK {
        bigint id PK
        bigint osiedle_id FK
        varchar imie
        varchar nazwisko
        varchar nr_mieszkania
        varchar email UK
        varchar haslo_hash
        boolean email_potwierdzony
        varchar rola "GOSC, MIESZKANIEC, MODERATOR"
        varchar status "OCZEKUJE, AKTYWNY, USUNIETY"
        timestamptz usuniety_at "anonimizacja RODO"
    }
    URZADZENIE {
        bigint id PK
        bigint uzytkownik_id FK
        varchar token_fcm UK
    }
    PRZEDMIOT {
        bigint id PK
        bigint osiedle_id FK
        bigint wlasciciel_id FK
        varchar nazwa
        text opis_stanu
        varchar kategoria
        boolean ukryty
        boolean archiwalny "soft-delete"
        text zdjecia_urls
    }
    OGLOSZENIE {
        bigint id PK
        bigint osiedle_id FK
        bigint autor_id FK
        text tresc
        timestamptz utworzone_at
    }
    WYPOZYCZENIE {
        bigint id PK
        bigint przedmiot_id FK
        bigint pozyczajacy_id FK "właściciel = blokada kalendarza"
        tstzrange okres "kiedy sprzęt jest zajęty"
        varchar status "OCZEKUJE, ZAAKCEPTOWANE, ODRZUCONE, ANULOWANE, WYDANE, ZAKONCZONE"
        boolean ocenione_przez_wlasciciela "weryfikacja double-blind"
        boolean ocenione_przez_pozyczajacego
        text zdjecia_wydania_urls
        text zdjecia_zwrotu_urls
    }
    KOD_POTWIERDZENIA {
        bigint id PK
        bigint wypozyczenie_id FK
        varchar etap "WYDANIE, ZWROT"
        char token_hash UK
        timestamptz wazny_do "np. 10 minut"
    }
    WIADOMOSC {
        bigint id PK
        bigint wypozyczenie_id FK
        bigint autor_id FK
        text tresc
        varchar zalacznik_url
        timestamptz wyslana_at
    }
    ZGLOSZENIE {
        bigint id PK
        bigint wypozyczenie_id FK
        bigint zglaszajacy_id FK
        varchar powod "USZKODZENIE, BRAK_ZWROTU, ZACHOWANIE"
        text opis
        text wyjasnienie "od drugiej strony"
        varchar decyzja "ZASADNE, NIEZASADNE"
        text uzasadnienie
        text zdjecia_urls
    }
    OCENA {
        bigint id PK
        bigint wypozyczenie_id FK
        bigint wystawiajacy_id FK
        bigint oceniany_id FK
        smallint gwiazdki "1-5"
        varchar komentarz
        boolean ukryta "odkrywana gdy obie strony ocenią"
    }
```

---

## 3. Więzy integralnościowe i indeksy
W modelu logicznym zaplanowano następujące mechanizmy bazodanowe dla spójności i wydajności zapytań:

| Tabela | Kolumna | Typ | Opis i uzasadnienie biznesowe |
| ----- | ----- | ----- | ----- |
| `UZYTKOWNIK` | `email` | `UNIQUE` | Zapewnia unikalność kont użytkowników – jeden adres e-mail może być przypisany tylko do jednego osiedla. |
| `KOD_ZAPROSZENIA` | `token_hash` | `UNIQUE` | Zabezpiecza przed duplikacją linków zaproszeń oraz jednorazowych kodów weryfikacyjnych. |
| `KOD_POTWIERDZENIA` | `token_hash` | `UNIQUE` | Zapewnia kryptograficzną unikalność kodu QR wykorzystywanego do fizycznego potwierdzania wydania lub zwrotu sprzętu. |
| `URZADZENIE` | `token_fcm` | `UNIQUE` | Unika duplikacji wysyłanych powiadomień push do tego samego urządzenia. |
| `OCENA` | `gwiazdki` | `CHECK` | Wymusza ograniczenie wartości w kolumnie wyłącznie do zakresu `1-5` na poziomie bazy. |
| `WYPOZYCZENIE` | `okres` | `EXCLUDE` | Ograniczenie typu *Exclusion Constraint* zapobiegające nakładaniu się terminów na siebie dla jednego przedmiotu. |
| `PRZEDMIOT` | `osiedle_id`, `kategoria` | `INDEX` (B-Tree) | Przyspiesza ładowanie i filtrowanie głównego katalogu sprzętu w aplikacji dla danego osiedla. |
| `WYPOZYCZENIE` | `pozyczajacy_id`, `status` | `INDEX` (B-Tree) | Optymalizuje zapytania zwracające historię oraz trwające rezerwacje zalogowanego użytkownika. |
| `WIADOMOSC` | `wypozyczenie_id`, `wyslana_at` | `INDEX` (B-Tree) | Przyspiesza chronologiczne ładowanie historii czatu dla konkretnego wypożyczenia. |

---

## 4. Przetwarzanie danych i specyfikacja technologiczna
Reprezentacja i komunikacja z danymi trwałymi opiera się o następujący stos technologiczny:

* **Baza danych:** Relacyjna baza danych **PostgreSQL (wersja 17)**, konfigurowana i uruchamiana w środowisku kontenerowym Docker.
* **Architektura serwera:** Aplikacja backendowa zaimplementowana w technologii **Spring Boot** (język Kotlin).
* **Komunikacja z bazą:** Połączenie z bazą odbywa się za pomocą natywnego protokołu **JDBC**.


---
