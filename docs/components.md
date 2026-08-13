# Компоненты Support AI

Из чего состоит решение — для обсуждения границ и переиспользования паттернов.

---

## Карта модулей

```text
app/
├── api/                          # HTTP-граница
│   ├── pachca.py                 # webhook, треды, follow-up
│   ├── pachca_buttons.py         # 👍 👎 🎫, локации, тикеты
│   ├── analytics.py              # дашборд, events, logs API
│   ├── llm.py                    # тестовый endpoint
│   └── health.py
│
├── services/                     # бизнес-логика
│   ├── dialogue_service.py       # диалог, уточнения, prompt
│   ├── wiki_service.py           # поиск, ranking, фильтры
│   ├── ticket_service.py         # создание тикетов
│   ├── tracker_service.py        # API Tracker
│   ├── pachca_service.py         # API Pachca
│   ├── pachca_poller.py          # локальный polling событий
│   ├── conversation_store.py     # контекст треда (JSON)
│   ├── feedback_service.py       # feedback memory
│   └── analytics_service.py      # метрики
│
├── knowledge/                    # конфигурация домена SD
│   ├── system_sd_matrix.py       # 150 систем → очередь
│   ├── tracker_routing.py        # keywords, routing, category
│   └── priorities.py             # приоритеты Wiki
│
├── web/
│   └── analytics_dashboard.py    # HTML-дашборд
│
├── analytics/                    # SQLite события
├── llm/                          # клиент LLM
└── core/
    ├── config.py
    └── log_buffer.py             # логи для дашборда
```

---

## Слои и ответственность

### 1. Канал — Pachca

| Компонент | Роль |
|-----------|------|
| `pachca_service` | Отправка сообщений, треды, кнопки |
| `pachca_poller` | Альтернатива webhook для localhost |
| `pachca.py` | Маршрутизация событий: message / button / thread |

### 2. Диалог — `dialogue_service`

- Уточняющие вопросы без LLM (VPN → ОС, login → сервис, Wi‑Fi → устройство)
- Сбор search query с учётом истории и последней ОС
- Формирование prompt с жёсткими правилами (только Wiki, одна ОС)
- Fallback «инструкция не найдена» → кнопка тикета

### 3. Retrieval — `wiki_service`

```text
Wiki API (до 50 hits)
  → score (title, slug, content)
  → priorities.py
  → feedback memory
  → фильтр ОС / темы / устройства
  → top 1–2 → LLM
```

Не использует vector DB — **keyword search Wiki + собственный ranking**.

### 4. Генерация — `llm`

- OpenAI-compatible API (корпоративный endpoint)
- Temperature 0.2, короткие ответы
- Запрет галлюцинаций через prompt + ограниченный context

### 5. ITSM — `ticket_service` + `tracker_routing`

| Шаг | Логика |
|-----|--------|
| Система SD | `detect_system()` по keywords |
| Очередь | матрица + локация SD |
| Компонент | маппинг SUPINT или «Другое» |
| Категория | `resolve_category()` по контексту диалога |
| Автор | login из Pachca → Tracker `author` |

### 6. Состояние диалога — `conversation_store`

Файл `data/conversations.json`:

- `interaction_id`, history, thread_chat_id
- флаг `action_taken` (один клик на кнопки)
- `awaiting_clarification`

### 7. Наблюдаемость

| Компонент | Данные |
|-----------|--------|
| `analytics_service` + SQLite | события: answer_sent, feedback, ticket_created |
| `log_buffer` | последние ~500 строк логов in-memory |
| `/analytics/summary` | HTML: метрики, routing tables, logs snapshot |

---

## Внешние зависимости

| Система | Назначение | Конфиг |
|---------|------------|--------|
| Pachca | Канал, кнопки, треды | `PACHCA_*` |
| Яндекс Wiki | База знаний | `WIKI_*` |
| Corporate LLM | Генерация ответа | `OPENAI_*` |
| Яндекс Tracker | Тикеты SD | `TRACKER_*` |

---

## Legacy / не в основном потоке

В репозитории есть заготовки agent/tools (`app/agents/`, `app/tools/`) — **основной production-поток идёт через `dialogue_service`**, не через autonomous agent. Это осознанное упрощение для предсказуемости и контроля.

---

## Переиспользуемые паттерны для других AI-решений

См. [standardization.md](standardization.md):

- Retrieval отдельно от generation
- Rule-based clarification до LLM
- Explicit domain config (`knowledge/`)
- Feedback loop + analytics
- Channel adapter (Pachca) отдельно от core
