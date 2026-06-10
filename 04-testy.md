# 04 — Scenariusze testów integracyjnych

---

## Test 1: Duplikat nie tworzy podwójnej epoki

1. Utwórz sesję, dostarcz wpis `(tl_A, seq=1)` → epoka = 1.
2. Dostarcz **ten sam** wpis ponownie.
3. **Oczekiwane:** epoka nadal = 1, brak duplikatu w historii.

---

## Test 2: Awaria przed zapisem nie psuje stanu

1. Dostarcz wpis, processor oblicza wynik.
2. Symuluj awarię przed COMMIT (przerwij transakcję).
3. Dostarcz ten sam wpis ponownie.
4. **Oczekiwane:** epoka = 1 (nie 0, nie 2). Stan spójny.

---

## Test 3: Dwie sesje z tym samym inputem dają ten sam BlueId

1. Dwie sesje z identycznym dokumentem początkowym.
2. Obie dostają te same wpisy, ale w różnej kolejności dotarcia.
3. **Oczekiwane:** po przetworzeniu wszystkich wpisów — identyczny BlueId w obu sesjach.

---

## Test 4: Dwa workery na jednej sesji — nie ma rozwidlenia

1. Worker 1 przetwarza wpis (trzyma lock).
2. Worker 2 próbuje przetworzyć kolejny wpis na tej samej sesji.
3. **Oczekiwane:** Worker 2 nie wchodzi — czeka aż Worker 1 skończy. Brak dwóch epok N+1.

---

## Test 5: Luka w timeline nie blokuje innych

1. Sesja subskrybuje timeline A i B.
2. Dostarcz `tl_A:1`, `tl_A:3` (brak seq=2), `tl_B:1`.
3. **Oczekiwane:** `tl_A:1` i `tl_B:1` przetworzone. `tl_A:3` czeka na `tl_A:2`.

---

## Test 6: Trujący wpis trafia do Dead Letter Queue , sesja kontynuuje

1. Dostarcz wpis z niepoprawnym payloadem → processor rzuca błąd.
2. System ponawia 3 razy, za każdym razem błąd.
3. Dostarcz poprawny wpis z innego timeline'a.
4. **Oczekiwane:** zły wpis w Dead Letter Queue, poprawny przetworzony normalnie, sesja ACTIVE.


## Środowisko dla testów 
- localstack/motoserver + lokalny postgres + docker images
- duplikat środowiska AWS + github actions

