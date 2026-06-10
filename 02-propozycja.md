## 2.1 Architektura wysokiego poziomu

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           WARSTWA TIMELINE'ÓW                               │
│                                                                             │
│   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                      │
│   │ Timeline A   │  │ Timeline B   │  │ Timeline C   │                      │
│   │ (Klient)     │  │ (Merchant)   │  │ (System)     │                      │
│   └──────┬───────┘  └──────┬───────┘  └──────┬───────┘                      │
│          │                  │                  │                            │
└──────────┼──────────────────┼──────────────────┼────────────────────────────┘
           │                  │                  │
           ▼                  ▼                  ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                     WARSTWA INGESTII (Ingestion)                            │
│                                                                             │
│   ┌─────────────────────────────────────────────────────────┐               │
│   │          Timeline Poller                                │               │
│   │  • Polling: GET /timelines/{id}/entries?after=seqNo     │               │
│   │  • Deduplikacja: UNIQUE(timeline_id, sequence_no)       │               │
│   │  • Output: wiadomość → SQS FIFO Queue                   │               │
│   └─────────────────────────┬───────────────────────────────┘               │
└─────────────────────────────┼───────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   ┌─────────────────────────────────────────────────────────┐               │
│   │               Session Worker (Lambda / ECS)             │               │
│   │                                                         │               │
│   │  1. Pozyskaj lock na sesję (SQS group + DB advisory)    │               │
│   │  2. Załaduj dokument sesji + wektor checkpointów        │               │
│   │  3. Checkpoint guard: seqNo > last? → proceed/skip      │               │
│   │  4. Wywołaj Blue Document Processor (deterministyczny)  │               │
│   │  5. Zweryfikuj BlueId outputu                           │               │
│   │  6. Atomic commit (epoch + checkpoint + outbox)         │               │
│   │  7. ACK wiadomość z kolejki                             │               │
│   └─────────────────────────┬───────────────────────────────┘               │
└─────────────────────────────┼───────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                     WARSTWA PERSYSTENCJI                                    │
│                                                                             │
│   ┌────────────────┐  ┌─────────────────┐  ┌────────────────┐               │
│   │  PostgreSQL    │  │  Event Outbox   │  │  S3            │               │
│   │  • Sesje       │  │  • Wyemitowane  │  │  • Snapshoty   │               │
│   │  • Epoki       │  │    zdarzenia    │  │    dokumentów  │               │
│   │  • Checkpointy │  │  • Relay → API  │  │  • Audit log   │               │
│   └────────────────┘  └─────────────────┘  └────────────────┘               │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Topologia AWS

```
API Gateway (REST) - Lambda (CRUD sesji)
EventBridge Scheduler - Lambda (Poller — co 5s per timeline)
SQS FIFO Queue - Lambda (Session Worker)
RDS PostgreSQL (Multi-AZ)
S3 (snapshoty dokumentów)
CloudWatch (metryki + logi + alarmy)
X-Ray (distributed tracing)
```



## Cykl życia sesji (Session Lifecycle)

```
          POST /sessions
               │
               ▼
     ┌──────────────────────────────────────┐
     │     ACTIVE                           │
     │                                      │
     │  • Konsumuje wpisy z timeline'ów     │
     │  • Przetwarza epoki                  │
     │  • Emituje zdarzenia                 │
     └────┬─────── ┬─────────────────────────┘
          │       │
 DELETE /sessions/{id}   błąd krytyczny
          │       │      (N nieudanych retry'ów)
          ▼       ▼
  ┌──────────────┐  ┌──────────┐
  │  TERMINATED  │  │  ERROR   │
  │              │  │          │
  │ • Zakończona │  │          │ 
  │ • Read-only  │  │          │
  │              │  │          │
  └──────────────┘  └──────────┘
```

## Przykładowy CRUD

| Metoda | Endpoint | Opis |
|--------|----------|------|
| `POST` | `/sessions` | Utwórz sesję z dokumentem początkowym + subskrypcjami |
| `GET` | `/sessions/{id}` | Status sesji, bieżąca epoka, BlueId |
| `GET` | `/sessions/{id}/epochs` | Lista epok (paginacja: `?after=epoch&limit=`) |
| `GET` | `/sessions/{id}/epochs/{n}` | Detale epoki: BlueId, entry_ref, gas, czas |
| `GET` | `/sessions/{id}/checkpoints` | Wektor checkpointów (per timeline) |
| `DELETE` | `/sessions/{id}` | Terminuj sesję |
 


## Model epok 

```
Epoch 0          Epoch 1          Epoch 2          Epoch 3
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│ Initial  │    │ Doc v1   │    │ Doc v2   │    │ Doc v3   │
│ Document │───▶│          │───▶│          │───▶│          │
│          │    │          │    │          │    │          │
│ BlueId₀  │    │ BlueId₁  │    │ BlueId₂  │    │ BlueId₃  │
└──────────┘    └──────────┘    └──────────┘    └──────────┘
                     ▲                ▲                ▲
                     │                │                │
              entry tl_A:1     entry tl_B:1     entry tl_A:2
              (customer pays)  (merchant ships) (customer confirms)
```

**Właściwości modelu epok:**

| Właściwość | Gwarancja |
|-----------|-----------|
| **Sekwencyjność** | Epoki rosną ściśle o 1: nie ma luk, nie ma skoków |
| **Determinizm** | BlueId epoki N jest funkcją: BlueId₀ + ordered_entries[1..N] |
| **Audytowalność** | Mając BlueId₀ i listę wpisów, można odtworzyć dowolną epokę od zera |
| **Atomowość** | Epoka albo jest w pełni zapisana (doc + checkpoint + outbox), albo nie istnieje |
| **Immutability** | Raz zapisana epoka nigdy się nie zmienia — append-only historia |

- PostgreSQL: `current_doc` (aktualny stan do szybkiego ładowania)
- S3: pełne snapshoty per epoka (audit trail; opcjonalnie, dla ważnych sesji)
- Replay: możliwy z `initial_doc` + sekwencja wpisów (nie wymaga snapshotów)

---


## Cykl życia Timeline Entry (Timeline-Entry Lifecycle)

```
┌────────────────────────────────────────────────────────────────────────┐
│                                                                        │
│  1. INGESTED                                                           │
│     Timeline provider dostarcza wpis (webhook/poll)                    │
│     → Deduplikacja: UNIQUE(timeline_id, sequence_no)                   │
│     → Duplikat? → 200 OK (idempotent, noop)                            │
│     → Nowy? → zapisz + wyślij do SQS FIFO                              │
│                         │                                              │
│  2. QUEUED              ▼                                              │
│     Wpis w SQS FIFO, MessageGroupId = sessionId                        │
│     DeduplicationId = "timelineId:sequenceNo"                          │
│     → SQS gwarantuje kolejność per group + dedup 5 min                 │
│                         │                                              │
│  3. DELIVERED           ▼                                              │
│     Worker odbiera wiadomość z SQS                                     │
│     → Ładuje sesję + checkpoint z DB                                   │
│     → Sprawdza: entry.seqNo > checkpoint.last_sequence_no?             │
│       • NIE → skip (already processed), ACK                            │
│       • TAK → przejdź do kroku 4                                       │
│                         │                                              │
│  4. PROCESSING          ▼                                              │
│     Worker wywołuje Blue Document Processor:                           │
│     processor.process(document, entry.message) → result                │
│     → Weryfikuj: compute_blue_id(result.document) = result.blueId?     │
│                         │                                              │
│  5. COMMITTED           ▼                                              │
│     Atomic transaction w PostgreSQL:                                   │
│       UPDATE session SET epoch=N+1, current_doc=..., current_doc_id=.. │
│       UPDATE checkpoint SET last_sequence_no = entry.seqNo             │
│       INSERT INTO epoch_history (...)                                  │
│       INSERT INTO event_outbox (...) — per emitted event               │
│     → ACK wiadomość w SQS                                              │
│                         │                                              │
│  6. EMITTED             ▼                                              │
│     Outbox relay (osobny Lambda/cron):                                 │
│     → POST emitted events na target timeline'y przez MyOS API          │
│     → Oznacz w outbox jako SENT                                        │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```



## Rozwiązanie problemu 1: Deterministyczna reguła scalania

```python
def kolejnosc_deterministyczna(wpisy: list[Wpis]) -> list[Wpis]:
    return sorted(wpisy, key=lambda w: (w.timestamp, w.timeline_id, w.sequence_no))
```

**Reguła:** Sortuj po `(timestamp, timeline_id, sequence_no)`.

**Gwarancje:**
1. **Determinizm:** Ten sam zbiór wpisów → zawsze ta sama kolejność, niezależnie od kolejności dotarcia.
2. **Zachowanie kolejności wewnątrz timeline'a:** Wpisy z jednego timeline'a zawsze w kolejności sequenceNo (bo timestamp rośnie monotonicznie per timeline).
3. **Przyczynowość:** Timestamp odzwierciedla przybliżoną kolejność czasową zdarzeń.
4. **Łamanie remisów:** `timeline_id` (sortowanie alfabetyczne) rozstrzyga wpisy z tym samym timestampem.

**Obsługa luk (gap handling):**
- Jeśli mamy wpis A:6 ale brakuje A:5 — wpis A:6 **czeka** aż A:5 dotrze.
- Wpisy z INNYCH timeline'ów mogą być przetwarzane w międzyczasie (nie blokują).
- Gwarantuje: wpisy z jednego timeline'a zawsze po kolei, bez przeskoków.

### Implementacja

Poller realizuje regułę scalania **w momencie wysyłki do kolejki**:
1. Poller pobiera nowe wpisy ze wszystkich subskrybowanych timeline'ów.
2. Sprawdza czy nie ma luk (wykrywanie braków).
3. Sortuje dostępne wpisy według reguły scalania.
4. Wysyła do SQS FIFO w tej kolejności (MessageGroupId = sessionId → zachowuje kolejność).
5. Worker przetwarza sekwencyjnie — nie musi znać reguły.

---


## Rozwiązanie problemu 2: Gwarancja przetworzenia „dokładnie raz"

Ten sam wpis może dotrzeć wielokrotnie (retry sieciowy, upłynięcie czasu widoczności w SQS, restart workera). System musi przetworzyć go dokładnie raz per sesja.

### Rozwiązanie: Trzy warstwy ochrony

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  WARSTWA 1: Deduplikacja na granicy ingestii                    │
│  ─────────────────────────────────────────────                  │
│  • SQS FIFO DeduplicationId = "timelineId:sequenceNo"           │
│  • Skutek: duplikat nigdy nie wchodzi do kolejki                │
│                                                                 │
│  WARSTWA 2: Strażnik checkpointu w workerze                     │
│  ─────────────────────────────────────────────                  │
│  • Przed przetworzeniem: wpis.seqNo > checkpoint.last_seq_no?   │
│  • Jeśli ≤ → wpis już przetworzony → pomiń + potwierdź          │
│  • Skutek: nawet jeśli SQS dostarczy ponownie, worker nie       │
│    przetworzy dwa razy                                          │
│                                                                 │
│  WARSTWA 3: Atomowy zapis (epoka + checkpoint + outbox)         │
│  ─────────────────────────────────────────────────              │
│  • Jedna transakcja PostgreSQL:                                 │
│    BEGIN;                                                       │
│      UPDATE document_sessions SET current_epoch=N+1, ...;       │
│      UPDATE session_checkpoints SET last_sequence_no=X;         │
│      INSERT INTO session_epochs (...);                          │
│      INSERT INTO event_outbox (...);                            │
│    COMMIT;                                                      │
│  • Skutek: awaria przed COMMIT → żadna zmiana nie widoczna;     │
│    ponowna próba jest bezpieczna (warstwa 2 przepuści)          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Dlaczego trzy warstwy?** Obrona w głąb — każda warstwa łapie inny scenariusz:
- Warstwa 1: zapobiega wejściu duplikatu do systemu
- Warstwa 2: chroni gdy duplikat przejdzie mimo warstwy 1 (edge case SQS)
- Warstwa 3: chroni przed stanem częściowym przy awarii

---



## Rozwiązanie problemu 3: Bezpieczeństwo wyścigów

Dwa workery jednocześnie przetwarzające tę samą sesję mogą spowodować rozwidlenie, utratę zapisu lub uszkodzenie checkpointu.

### Rozwiązanie: Dwupoziomowa gwarancja jednego pisarza

| Poziom | Mechanizm | Chroni przed |
|--------|-----------|-------------|
| 1 (główny) | **SQS FIFO MessageGroupId = sessionId** — kolejka dostarcza maks. 1 wiadomość na sesję naraz | Dwóch workerów przetwarzających jednocześnie |
| 2 (siatka bezpieczeństwa) | **PostgreSQL advisory lock:** `SELECT pg_try_advisory_xact_lock(hashtext(session_id))` | Sytuacja brzegowa: upłynął czas widoczności w SQS zanim worker skończył → ponowne dostarczenie |

**Tabela ochrony w różnych scenariuszach:**

| Scenariusz | SQS FIFO | DB Lock | Checkpoint |
|------------|----------|---------|------------|
| Normalny przepływ | ✅ 1 wiadomość naraz | Nie potrzebny | Nie potrzebny |
| Upłynął czas widoczności | ❌ puszcza drugiego | ✅ blokuje | ✅ pomija jeśli już przetworzone |
| Awaria workera przed zapisem | ✅ ponowne dostarczenie | Zwolniony automatycznie | ✅ seqNo nie przesunięty |
| Awaria workera po zapisie | ✅ ponowne dostarczenie | Zwolniony automatycznie | ✅ seqNo > last → pomiń |

**Dlaczego oba poziomy?**
- SQS FIFO wystarczy w 99.9% przypadków (zero dodatkowego kosztu).
- Advisory lock chroni przed jedynym edge case'em SQS: upłynięcie czasu widoczności podczas długiego przetwarzania BEX.

---


## Rozwiązanie problemu 4: Dowód zbieżności


Dwie niezależne instancje sesji z tym samym dokumentem i wpisami MUSZĄ dojść do identycznego BlueId. Jak to zagwarantować i weryfikować?

### Rozwiązanie: Weryfikacja BlueId po każdej epoce

**Warunki wystarczające dla konwergencji:**

1. ✅ Ten sam dokument początkowy (identyczny BlueId₀)
2. ✅ Te same wpisy timeline'ów (identyczne payloady + sequenceNo)
3. ✅ Ta sama kolejność przetwarzania (gwarantowana przez regułę scalania — §2.7)
4. ✅ Deterministyczny processor (BEX — brak losowości, brak zegara, brak sieci)

**Aktywna weryfikacja w czasie działania:**

Po każdej epoce system oblicza BlueId nowego dokumentu i sprawdza:
- Czy BlueId zgadza się z wynikiem zwróconym przez processor?
- Opcjonalnie: czy niezależna instancja (cień) dochodzi do tego samego BlueId?

```
Po przetworzeniu epoki N:
  actual_id   = oblicz_blue_id(nowy_dokument)
  expected_id = processor.wynik.blueId
  
  jeśli actual_id ≠ expected_id:
      → ALARM KRYTYCZNY
      → Zamroź sesję (status = ERROR)
      → Nie zapisuj epoki
```

**Co jeśli konwergencja jest naruszona?**
- Sesja przechodzi w status ERROR (nie zapisuje wadliwej epoki).
- Alert krytyczny do operatora.
- Konieczna analiza: bug w procesorze? Uszkodzenie danych?
- Sesję można odbudować od zera (replay z BlueId₀ + wpisy) po naprawieniu buga.

---
