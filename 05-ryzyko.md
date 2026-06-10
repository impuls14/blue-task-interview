# 05 — Ryzyka, ograniczenia i przyszłe usprawnienia

---

## Ryzyka

| Ryzyko | Wpływ | Mitigacja |
|--------|-------|-----------|
| Bug w procesorze łamie determinizm | Krytyczny — sesje się rozjeżdżają | Weryfikacja BlueId po każdej epoce; alert natychmiast |
| Poller nie nadąża z pobieraniem wpisów | Rosnące opóźnienie | Monitoring driftu; skalowanie pollerów |
| Trujący wpis blokuje timeline | Jedna sekwencja stoi | Kolejka błędnych wiadomości po 3 próbach; inne timeline'y działają dalej |
| Awaria bazy danych | System nie przetwarza | RDS Multi-AZ; automatyczny failover |
| Złożoność rozproszonej architektury AWS | Trudniejsze debugowanie, wiele punktów awarii (Lambda, SQS, RDS, S3), zależność od dostępności usług AWS | Observability (logi, metryki, tracing); testy integracyjne na localstack; runbooki na wypadek awarii |

---

## Co design gwarantuje

- Przetworzenie każdego wpisu dokładnie raz per sesja
- Deterministyczną kolejność wpisów z wielu timeline'ów
- Zbieżność niezależnych sesji do tego samego BlueId
- Bezpieczeństwo przy awariach (brak częściowego stanu)
- Izolację błędów (jeden zły wpis nie psuje całości)

## Czego design NIE gwarantuje

- Opóźnienie poniżej 100ms (polling co 5s + Lambda cold start)
- Wysoką dostępność między regionami (v1 = jeden region)
- Dokładnie jednokrotną publikację zdarzeń na zewnątrz (outbox relay = co najmniej raz)

---

## Przyszłe usprawnienia

- **Niższe opóźnienie:** Zamiana Lambda na kontener ECS z ciągłym nasłuchiwaniem (reakcja w milisekundach zamiast sekund)
- **Szybsza publikacja zdarzeń:** Nasłuchiwanie na zmiany w bazie zamiast odpytywania co 5s
- **Skalowalność:** Podział sesji na wiele instancji bazy danych (sharding) gdy jedna baza nie wystarczy
- **Formalna weryfikacja:** Matematyczny dowód poprawności reguły scalania (nie tylko testy)
