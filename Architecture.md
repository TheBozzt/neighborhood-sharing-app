**PZSP2, zespół:** Mateusz Hałub, Mateusz Duda, Wojciech Szozda, Piotr Winiarczyk
## Materiał na konsultacje z architektury:

| Element | Technologia |
| --- | --- |
| Aplikacja mobilna | Kotlin |
| Serwer aplikacyjny | Kotlin, Spring Boot |
| Baza danych | PostgreSQL |
| Magazyn zdjęć | Blob - PostgreSQL |
| Powiadomienia push | Firebase Cloud Messaging |
| Wdrożenie | Docker Compose na jednej maszynie wirtualnej |


## REST API

Wszystkie ścieżki z prefiksem `/api`. Kolumna „Rola”: G – gość, M – mieszkaniec, Mod – moderator.

| Metoda i ścieżka | Opis | Rola |
| --- | --- | --- |
| `POST /auth/rejestracja` | założenie konta (z kodem osiedla, zgody) | publiczne |
| `POST /auth/potwierdzenie-email` | jednorazowy kod z e-maila | G |
| `POST /auth/logowanie`  | tokeny JWT | wszyscy |
| `GET /user/{id}`, `PATCH /user/{id}`, `DELETE /user/{id}` | profil, status wniosku, edycja, usunięcie konta | G, M |
| `GET /przedmioty?q=&kategoria=&od=&do=` | katalog z wyszukiwaniem i filtrem dostępności | M |
| `POST /przedmioty`, `PATCH /przedmioty/{id}`, `DELETE /przedmioty/{id}` | własne przedmioty, ukrywanie | M |
| `GET /przedmioty/{id}/kalendarz?tydzien=` | zajęte i wolne godziny | M |
| `POST /rezerwacje` | prośba o rezerwację | M |
| `POST /rezerwacje/{id}/akceptacja`, `/odrzucenie`, `/anulowanie` | decyzja właściciela, anulowanie | M |
| `POST /wypozyczenia/{id}/wydanie` | zdjęcia + wygenerowanie kodu QR wydania | M (właściciel) |
| `POST /wypozyczenia/{id}/wydanie/potwierdzenie` | zeskanowany token | M (pożyczający) |
| `POST /wypozyczenia/{id}/zwrot`, `/zwrot/potwierdzenie` | jak wyżej, role odwrócone | M |
| `GET /wypozyczenia?rola=&status=` | historia wypożyczeń | M |
| `GET, POST /wypozyczenia/{id}/wiadomosci` | czat | M (strony) |
| `POST /wypozyczenia/{id}/oceny` | ocena drugiej strony | M (strony) |
| `GET /uzytkownicy/{id}/oceny` | oceny i średnia | M |
| `POST /wypozyczenia/{id}/zgloszenia` | zgłoszenie problemu | M (strony) |
| `POST /zgloszenia/{id}/wyjasnienie` | wyjaśnienie drugiej strony | M |
| `GET /zgloszenia`, `GET /zgloszenia/{id}`, `POST /zgloszenia/{id}/decyzja` | widok i decyzja moderatora | Mod |
| `GET, POST /ogloszenia`, `DELETE /ogloszenia/{id}` | tablica ogłoszeń, moderacja | M, Mod |
| `POST /osiedla`, `PATCH /osiedla/{id}` | tworzenie i edycja osiedla | Mod |
| `POST /osiedla/{id}/kody`, `DELETE /osiedla/{id}/kody/{kodId}`, `GET /osiedla/{id}/kody/{kodId}/pdf` | kody QR i zaproszenia | Mod |
| `DELETE /osiedla/{id}/czlonkowie/{uzytkownikId}` | usunięcie użytkownika | Mod
| `GET /osiedla/{id}/statystyki?od=&do=` | statystyki | Mod


---


## Diagram wdrożenia
```mermaid
flowchart LR
    %% Definicja stylów dla notacji zbliżonej do C4
    classDef person fill:#08427b,stroke:#052e56,color:#fff
    classDef container fill:#1168bd,stroke:#0b4884,color:#fff
    classDef external fill:#999999,stroke:#666666,color:#fff
    classDef boundary fill:none,stroke:#444,stroke-dasharray:5 5,stroke-width:2px

    User(("Mieszkaniec\n(Użytkownik)")):::person

    subgraph Phone ["Urządzenie: Telefon z Androidem"]
        App["Aplikacja Mobilna\n[Kotlin, Jetpack Compose]"]:::container
    end

    subgraph VM ["Infrastruktura: Maszyna Wirtualna (Docker Compose)"]
        Backend["Serwer Aplikacyjny\n[Spring Boot, Kotlin]\nAPI, Logika biznesowa, JWT"]:::container
        DB[("Baza Danych\n[PostgreSQL]\nDane i Blob zdjęć")]:::container
    end

    FCM["Firebase Cloud Messaging\n[Usługa Zewnętrzna]"]:::external

    User -->|Używa interfejsu| App
    App -->|Wysyła żądania HTTPS/REST| Backend
    Backend -->|Zapisuje/Odczytuje JDBC| DB
    Backend -->|Zleca powiadomienia HTTPS| FCM
    FCM -.->|Wysyła push| App

    class Phone,VM boundary

```


## Perspektywa logiczna

Perspektywa logiczna składa się z trzech diagramów: (a) moduły backendu i zależności między nimi, (b) dane, za które odpowiada każdy moduł, (c) sekwencja przypadku użycia „Wypożyczenie sprzętu”.

### a) Moduły i zależności

```mermaid
flowchart LR
    classDef component fill:#85bbf0,stroke:#5b82a8,color:#000
    classDef external fill:#999999,stroke:#666666,color:#fff
    classDef boundary fill:none,stroke:#444,stroke-dasharray:5 5,stroke-width:2px

    App["Aplikacja mobilna<br/>[Kotlin]"]:::external
    FCM["Firebase Cloud Messaging<br/>[usługa zewnętrzna]"]:::external

    subgraph Backend ["Backend (Spring Boot)"]
        direction TB
        API["Kontrolery REST API<br/>(/api/...)"]:::component
        Auth["Moduł Autoryzacji<br/>(rejestracja, logowanie JWT, role)"]:::component
        Katalog["Moduł Katalogu<br/>(przedmioty, wyszukiwanie, kalendarz)"]:::component
        Rezerwacje["Moduł Wypożyczeń<br/>(rezerwacje, wydanie i zwrot z kodem QR)"]:::component
        Komunikacja["Moduł Komunikacji<br/>(czat, oceny, zgłoszenia, tablica)"]:::component
        Moderacja["Moduł Moderacji<br/>(osiedla, kody zaproszeń, decyzje, statystyki)"]:::component
        Powiadomienia["Moduł Powiadomień<br/>(tokeny urządzeń, wysyłka push)"]:::component
        Pliki["Moduł Plików<br/>(zdjęcia jako Blob)"]:::component

        API --> Auth
        API --> Katalog
        API --> Rezerwacje
        API --> Komunikacja
        API --> Moderacja

        Katalog -->|zalogowany użytkownik, osiedle| Auth
        Rezerwacje -->|przedmiot i jego kalendarz| Katalog
        Komunikacja -->|strony wypożyczenia| Rezerwacje
        Moderacja -->|decyzje w zgłoszeniach| Komunikacja
        Moderacja -->|członkowie osiedla| Auth

        Rezerwacje --> Powiadomienia
        Komunikacja --> Powiadomienia
        Katalog --> Pliki
        Rezerwacje --> Pliki
        Komunikacja --> Pliki
    end

    App -->|HTTPS / REST, token JWT| API
    Powiadomienia -->|HTTPS| FCM
    FCM -.->|push| App

    class Backend boundary
```

### b) Moduły i dane

```mermaid
flowchart LR
    classDef component fill:#85bbf0,stroke:#5b82a8,color:#000
    classDef table fill:#ffffff,stroke:#5b82a8,color:#000
    classDef boundary fill:none,stroke:#444,stroke-dasharray:5 5,stroke-width:2px

    subgraph Moduly ["Moduły backendu"]
        direction TB
        Auth["Moduł Autoryzacji"]:::component
        Moderacja["Moduł Moderacji"]:::component
        Katalog["Moduł Katalogu"]:::component
        Rezerwacje["Moduł Wypożyczeń"]:::component
        Komunikacja["Moduł Komunikacji"]:::component
        Powiadomienia["Moduł Powiadomień"]:::component
        Pliki["Moduł Plików"]:::component
    end

    subgraph DB ["Baza danych (PostgreSQL)"]
        direction TB
        T_UZ[("UZYTKOWNIK")]:::table
        T_OS[("OSIEDLE")]:::table
        T_KZ[("KOD_ZAPROSZENIA")]:::table
        T_PR[("PRZEDMIOT")]:::table
        T_WY[("WYPOZYCZENIE")]:::table
        T_KP[("KOD_POTWIERDZENIA")]:::table
        T_WI[("WIADOMOSC")]:::table
        T_OC[("OCENA")]:::table
        T_ZG[("ZGLOSZENIE")]:::table
        T_OG[("OGLOSZENIE")]:::table
        T_UR[("URZADZENIE")]:::table
        T_BL[("Zdjęcia (Blob)")]:::table
    end

    Auth --> T_UZ
    Auth -.->|weryfikacja kodu przy rejestracji| T_KZ
    Moderacja --> T_OS
    Moderacja --> T_KZ
    Katalog --> T_PR
    Rezerwacje --> T_WY
    Rezerwacje --> T_KP
    Komunikacja --> T_WI
    Komunikacja --> T_OC
    Komunikacja --> T_ZG
    Komunikacja --> T_OG
    Powiadomienia --> T_UR
    Pliki --> T_BL

    class Moduly,DB boundary
```

### c) Diagram sekwencji – przypadek użycia „Wypożyczenie sprzętu”

```mermaid
sequenceDiagram
    autonumber
    actor P as Pożyczający (aplikacja)
    actor W as Właściciel (aplikacja)
    participant API as Backend (REST API)
    participant DB as Baza danych
    participant FCM as Firebase Cloud Messaging

    Note over P,DB: Rezerwacja
    P->>API: GET /przedmioty/{id}/kalendarz?tydzien=
    API->>DB: Pobierz zajęte terminy przedmiotu
    DB-->>API: Lista terminów
    API-->>P: Kalendarz tygodniowy
    P->>API: POST /rezerwacje (przedmiotId, od, do)
    API->>DB: Sprawdź kolizję i zapisz WYPOZYCZENIE (OCZEKUJE)
    DB-->>API: OK
    API-->>P: 201 Created – prośba wysłana
    API->>FCM: Powiadomienie „Nowa prośba o rezerwację”
    FCM-->>W: Push

    Note over W,DB: Decyzja właściciela
    W->>API: POST /rezerwacje/{id}/akceptacja
    API->>DB: Zmień status na ZAAKCEPTOWANE
    alt Termin wolny
        DB-->>API: OK (EXCLUDE nie zgłasza konfliktu)
        API-->>W: 200 OK
        API->>FCM: Powiadomienie „Rezerwacja zaakceptowana”
        FCM-->>P: Push
    else Termin zajęty w międzyczasie
        DB-->>API: Naruszenie EXCLUDE
        API-->>W: 409 TERMIN_ZAJETY
    end

    Note over P,W: Wydanie sprzętu przy spotkaniu
    W->>API: POST /wypozyczenia/{id}/wydanie (zdjęcia stanu)
    API->>DB: Zapisz zdjęcia i KOD_POTWIERDZENIA (hash, ważny 10 min)
    API-->>W: Kod QR
    P->>P: Skan kodu QR
    P->>API: POST /wypozyczenia/{id}/wydanie/potwierdzenie (kod)
    API->>DB: Sprawdź kod, status WYDANE, usuń kod
    API-->>P: 200 OK – sprzęt odebrany

    Note over P,W: Zwrot sprzętu (role odwrócone)
    P->>API: POST /wypozyczenia/{id}/zwrot (zdjęcia stanu)
    API->>DB: Zapisz zdjęcia i KOD_POTWIERDZENIA
    API-->>P: Kod QR
    W->>API: POST /wypozyczenia/{id}/zwrot/potwierdzenie (kod)
    API->>DB: Sprawdź kod, status ZAKONCZONE
    API-->>W: 200 OK – zwrot potwierdzony
    API->>FCM: Powiadomienie „Oceń wypożyczenie”
    FCM-->>P: Push
```