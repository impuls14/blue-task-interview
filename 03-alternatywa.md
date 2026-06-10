# 03 — Rozważone alternatywy

Odrzucone podejścia dla kluczowych decyzji wpływających na poprawność i niezawodność.

---

## Kolejność wpisów z wielu timeline'ów

| Alternatywa | Odrzucone bo |
|-------------|-------------|
| Kolejność dotarcia (kto pierwszy ten lepszy) | Niedeterministyczna — dwie instancje mogą widzieć inną kolejność i dojść do różnych stanów |
| Na zmianę z każdego timeline'a (po jednym z każdego) | Zależy od tego ile wpisów dotarło w danym momencie; wynik niestabilny |
| Logiczny zegar rozproszony (wymaga współpracy uczestników) | Każdy uczestnik musiałby koordynować numerację z innymi; zbyt złożone i nierealistyczne dla v1 |

---

## Gwarancja jednego pisarza na sesję

| Alternatywa | Odrzucone bo |
|-------------|-------------|
| Optymistyczne blokowanie (zapisz i sprawdź czy nikt nie zmienił w międzyczasie) | Przy kolizji marnuje obliczenia processora — już policzył wynik, a musi go wyrzucić i liczyć od nowa |
| Osobny serwer blokad (np. Redis) | Dodaje zbędny komponent do utrzymania; blokada w PostgreSQL daje to samo bez dodatkowej infrastruktury |

---

## Baza danych

| Alternatywa | Odrzucone bo |
|-------------|-------------|
| DynamoDB (baza klucz-wartość od AWS) | Nie obsługuje transakcji obejmujących wiele tabel naraz — a ja potrzebuję atomowo zapisać epokę + checkpoint + zdarzenia do wyemitowania |
| Specjalizowana baza zdarzeń (EventStoreDB) | Niszowy produkt; brak zarządzanej usługi AWS; przesada dla pierwszej wersji |

---

## Publikacja wyemitowanych zdarzeń

| Alternatywa | Odrzucone bo |
|-------------|-------------|
| Bezpośrednia publikacja w trakcie przetwarzania | Problem podwójnego zapisu: jeśli wysłanie zdarzenia przejdzie ale zapis do bazy się wycofa → w systemie pojawi się „fantomowe" zdarzenie którego nigdy nie było |
| Nasłuchiwanie na zmiany w bazie danych (Debezium/DMS) | Wymaga dodatkowej infrastruktury do utrzymania; przesada dla pierwszej wersji |

---

## Odkrywanie nowych wpisów

| Alternatywa | Odrzucone bo |
|-------------|-------------|
| Tylko webhook (czekaj aż dostawca powiadomi) | Brak mechanizmu nadrabiania po awarii — jeśli webhook nie dotrze, system nie wie o nowym wpisie |
| Webhook + odpytywanie jednocześnie | Dodaje złożoność obsługi dwóch źródeł naraz; webhook jako rozszerzenie w przyszłej wersji |
