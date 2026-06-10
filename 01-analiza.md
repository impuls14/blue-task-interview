# 01 — Analiza

Blue Document Processor to silnik deterministyczny: przyjmuje dokument i zdarzenie, dopasowuje zdarzenie do kanału (kontraktu), uruchamia wewnętrzną logikę (BEX), aplikuje zmiany (patche) i produkuje nowy kanoniczny dokument z weryfikowalnym hashem (BlueId).

Równanie przetwarzania:

```
dokument[epoch N] + zdarzenie + kontrakty + BEX
= dokument[epoch N+1] + wyemitowane zdarzenia + gas + checkpoint
```

Problem: Zaprojektować system który uruchamia i zarządza sesjami dokumentów Blue, gdzie każda sesja konsumuje wpisy z wielu niezależnych timeline'ów. System musi gwarantować deterministyczną konwergencję, idempotentność, bezpieczeństwo przy współbieżności i odporność na awarie — w środowisku rozproszonym z semantyką at-least-once delivery.


## Kluczowe pojęcia domeny

| Pojęcie | Definicja |
|---------|-----------|
| **Timeline** | Append-only log należący do jednego konta. Tylko właściciel dopisuje wpisy; inni mogą czytać. Każdy wpis ma monotonicznie rosnący `sequenceNo`. |
| **Timeline Entry** | Niezmienny rekord: `(timelineId, sequenceNo, message, timestamp)`. |
| **Sesja dokumentu (Document Session)** | Uruchomiony proces trzymający dokument Blue i konsumujący wpisy z subskrybowanych timeline'ów. |
| **Epoka (Epoch)** | Wersja dokumentu sesji po każdym udanym przetworzeniu wpisu. Epoka 0 = dokument początkowy. |
| **Checkpoint** | Wewnątrz-dokumentowy rekord per kanał, przechowujący identyfikator ostatniego przetworzonego wpisu — zapobiega podwójnemu przetworzeniu. |
| **Kanał (Channel)** | Kontrakt w dokumencie definiujący z którego timeline'a i jakie typy wiadomości dokument akceptuje. |
| **Operacja (Operation)** | Wpis timeline'a typu `Operation Request`, pasujący do kontraktu i wyzwalający workflow. |
| **BlueId** | SHA-256/Base58 hash kanonicznej postaci dokumentu — kryptograficzny dowód stanu. |
| **BEX** | Deterministyczny język wyrażeń wykonywany wewnątrz dokumentu: zero side-effectów, brak dostępu do sieci/czasu/losowości. |

---

## Problemy 

### Problem 1: Deterministyczna reguła scalania

Każdy timeline posiada własne niezależne numerowanie (`sequenceNo`). Nie istnieje globalny zegar ani globalny sekwencer łączący timeline'y. Musimy zdefiniować deterministyczną regułę scalania — politykę, która dla dowolnego zbioru wpisów z wielu timeline'ów zawsze wyprodukuje tę samą kolejność przetwarzania.

Jeśli dwie instancje systemu widzą wpisy w różnej kolejności dostarczenia, ale stosują tę samą regułę scalania, muszą dojść do identycznej sekwencji. Reguła musi być:
- Niezależna od kolejności dotarcia wpisów do systemu
- Deterministyczna (te same dane → ta sama kolejność, zawsze)
- Zachowująca kolejność wewnątrz pojedynczego timeline'a (wpis 5 przed wpisem 6)

### Problem 2: Gwarancja przetworzenia „dokładnie raz"

Dostawcy timeline'ów (MyOS API, webhooki) gwarantują co najwyżej dostarczenie „co najmniej raz" (at-least-once). Ten sam wpis może dotrzeć do systemu wielokrotnie z powodu:
- Powtórzeń sieciowych (retry)
- Upłynięcia limitu czasu na webhook (dostawca nie otrzymał potwierdzenia → wysyła ponownie)
- Restartu workera po awarii (upłynął czas widoczności w SQS → ponowne dostarczenie)

System musi przetworzyć każdy wpis dokładnie raz na sesję, niezależnie od liczby dostarczeń.

### Problem 3: Współbieżne przetwarzanie sesji

W architekturze z wieloma workerami, dwa procesy mogą próbować jednocześnie przetworzyć tę samą sesję. Skutki:
- Rozwidlenie: Dwa workery produkują dwie różne epoki N+1 (sprzeczne stany)
- Utracony zapis (lost update): Jeden worker nadpisuje wynik drugiego
- Uszkodzenie checkpointu: Checkpoint przesunięty bez faktycznie utrwalonej epoki

Wymaga gwarancji jednego pisarza na sesję w środowisku rozproszonym.

### Problem 4: Dowód zbieżności

Fundamentalny wymóg: dwie niezależne instancje sesji (np. replika cienia, odbudowa po awarii) z tym samym dokumentem początkowym i tymi samymi wpisami muszą dojść do identycznego BlueId. Jeśli kiedykolwiek ich stany się rozjadą — system jest zepsuty.

Wymaga weryfikacji w czasie działania (nie wystarczy „zaprojektować poprawnie"; trzeba aktywnie sprawdzać).

---

