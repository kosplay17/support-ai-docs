# Архитектура Support AI

## Назначение

Support AI снижает нагрузку на Service Desk за счёт **самообслуживания по базе знаний** и **автоматического оформления обращений**, когда инструкции недостаточно.

Ключевой принцип: **LLM не является источником истины** — ответ строится только на найденных инструкциях Wiki + контексте диалога.

---

## Общая схема

```mermaid
flowchart TB
    subgraph Users["Пользователи"]
        U[Сотрудник Lamoda]
    end

    subgraph Channel["Канал"]
        P[Pachca — канал / тред]
    end

    subgraph SupportAI["Support AI — FastAPI"]
        WH["/pachca/webhook"]
        POL[Pachca Poller]
        DS[dialogue_service]
        WS[wiki_service]
        LLM[Corporate LLM]
        TS[ticket_service]
        RT[tracker_routing]
        AN[analytics + logs]
    end

    subgraph Knowledge["База знаний"]
        W[Яндекс Wiki cf/]
    end

    subgraph ITSM["Service Desk"]
        TR[Яндекс Tracker]
    end

    U --> P
    P --> WH
    POL -.->|локально| WH
    WH --> DS
    DS --> WS
    WS --> W
    DS --> LLM
    WH --> TS
    TS --> RT
    TS --> TR
    WH --> AN
```

---

## Слои решения

```text
┌─────────────────────────────────────────────────────────┐
│  Канал взаимодействия          Pachca (webhook/poller)  │
├─────────────────────────────────────────────────────────┤
│  API-слой                      app/api/                 │
├─────────────────────────────────────────────────────────┤
│  Диалог и оркестрация          dialogue_service           │
├─────────────────────────────────────────────────────────┤
│  Retrieval (поиск)             wiki_service + priorities  │
├─────────────────────────────────────────────────────────┤
│  Генерация ответа              llm (OpenAI-compatible)  │
├─────────────────────────────────────────────────────────┤
│  Маршрутизация SD              tracker_routing + matrix   │
├─────────────────────────────────────────────────────────┤
│  Тикеты                        ticket_service + tracker   │
├─────────────────────────────────────────────────────────┤
│  Наблюдаемость                 analytics, log_buffer    │
└─────────────────────────────────────────────────────────┘
```

Подробнее по модулям: [components.md](components.md).  
Визуальные схемы: [diagrams/support-ai-architecture.drawio](diagrams/support-ai-architecture.drawio) · [PNG](diagrams/support-ai-architecture.png)

---

## Поток: новое обращение

```mermaid
sequenceDiagram
    participant U as Пользователь
    participant P as Pachca
    participant API as Support AI
    participant W as Wiki
    participant L as LLM
    participant T as Tracker

    U->>P: Сообщение в канал
    P->>API: webhook message_new
    API->>P: create_thread
    API->>API: needs_clarification?<br/>(ОС, сервис, устройство)
    alt Нужно уточнение
        API->>P: Вопрос в тред (без кнопок)
    else Достаточно контекста
        API->>W: search + ranking
        W-->>API: top 1–2 инструкции
        API->>L: prompt + инструкции
        L-->>API: ответ
        API->>P: Ответ + кнопки 👍 👎 🎫
    end
    U->>P: 👎 или 🎫 + локация SD
    P->>API: button click
    API->>T: create issue (author, routing, dialog)
    API->>P: Ссылка на тикет
```

---

## Поток: уточнение в треде

```mermaid
sequenceDiagram
    participant U as Пользователь
    participant P as Pachca
    participant API as Support AI
    participant Store as conversation_store

    U->>P: «виндовс» (в треде)
    P->>API: webhook (thread.message_chat_id)
    API->>Store: найти контекст диалога
    API->>API: ask(question, history)
    API->>P: Ответ по VPN Windows + кнопки
```

Важно: для сообщений в треде контекст ищется по `thread.message_chat_id`, а не только по `chat_id` канала.

---

## Поток: маршрутизация тикета

```text
Диалог + локация SD
        │
        ├── Диагностика ИТ (Wi‑Fi, кнопки) → без systemsd, компонент «Диагностика»
        │
        ├── Keywords → Система SD (149 систем)
        │       → очередь SUPINT / TECHCX / по локации
        │       → компонент SUPINT (или «Другое»)
        │
        └── Не распознано → «Нет в списке (другой)», TECHCX

Категория обращения — по контексту диалога (resolve_category), не по жёсткой таблице.
```

Матрица: `app/knowledge/system_sd_matrix.py` (150 систем SD).

---

## Принципы архитектуры

| Принцип | Реализация |
|---------|------------|
| **Grounded answers** | LLM получает только top 1–2 страницы Wiki |
| **Минимум LLM-вызовов** | Уточнения (ОС, сервис) — rule-based, без LLM |
| **Explicit routing** | Keywords + матрица SD, не «магия LLM» |
| **Human in the loop** | 👍/👎, кнопка тикета, автор тикета = пользователь |
| **Observability** | SQLite-события, дашборд, in-memory логи |
| **Итеративная калибровка** | Keywords и категории — по реальным обращениям |

---

## Что сознательно не в scope v0.6

- Vector DB / embeddings (поиск через Wiki API + ranking)
- Autonomous agent с произвольными tool calls
- Multi-channel (только Pachca)
- Автотесты маршрутизации (калибровка в боевой среде)

---

## Связанные документы

- [components.md](components.md)
- [integrations.md](integrations.md)
- [roadmap.md](roadmap.md)
