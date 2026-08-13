> Детальный runbook (скрипты, env, команды разработки) — во **внутреннем репозитории** Support AI.
>
# Эксплуатация

Запуск, мониторинг и итерация Support AI в production.

---

## Запуск

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env   # заполнить токены
./scripts/start_server.sh   # uvicorn + watchdog
```

**Не используйте `--reload`** — на macOS reloader часто оставляет процесс без worker.

`start_server.sh` поднимает **uvicorn и watchdog**. Watchdog каждые 15 сек проверяет `/health`.

### Burst в канале (debounce)

Подряд идущие сообщения **одного автора в канале** склеиваются (по умолчанию **5 сек**) — один ответ на пакет, тред на **последнее** сообщение.

Если пауза > окна burst, но < **1 мин** (`CHANNEL_COALESCE_SEC`), следующие посты идут в **уже открытый тред** только если бот **ждёт уточнение** или ещё формирует первый ответ. **Новая проблема** — новый тред, даже внутри минуты.

- Env: `PACHCA_CHANNEL_BURST_SEC=5.0` (0 = выключить)
- Env: `CHANNEL_COALESCE_SEC=60`
- Env: `CLARIFY_USE_LLM=true` — первый ход: LLM решает, чего не хватает (fallback — правила; greeting / human summon / переговорка — всегда правила)
- Работает в **poller** и при прямом **webhook**
- Follow-up в треде и кнопки — без debounce

### TTL сессий

- `CONVERSATION_TTL_HOURS=24` — follow-up не резолвится в «мёртвые» треды; после тикета бот слушает тред ещё до 24ч
- `CONVERSATION_CLOSE_AFTER_TICKET=false` — сессия **не** закрывается после создания тикета (статус по известному ключу в треде без повторного вопроса)
- При старте сервера — `prune` старых записей в `data/conversations.json`

**Проверка статуса:**

```bash
./scripts/status.sh
```

Логи: `/tmp/support-ai-watchdog.log`, `/tmp/support-ai-uvicorn.log`

### v1.0 — локальный чеклист

| Шаг | Команда / URL |
|-----|----------------|
| Статус | `./scripts/status.sh` |
| Smoke сценариев | `python scripts/scenario_smoke.py` |
| KPI | http://127.0.0.1:8000/analytics/summary |
| Gaps Wiki | http://127.0.0.1:8000/learning/report |
| Тест в Pachca | сообщение в канал → «Обновить» на дашборде |

Ориентиры KPI на дашборде: Deflection ≥30%, без инструкции ≤25%, ошибки ответа = 0.

### Watchdog (отдельный запуск, если нужен)

```bash
nohup python scripts/watchdog.py >> /tmp/support-ai-watchdog.log 2>&1 &
```

Обычно не нужен — `start_server.sh` уже стартует watchdog.

### Нагрузочный тест

```bash
python scripts/load_test.py
python scripts/load_test.py --work-ms 3000 --concurrency 1,3,5,8,10
```

Симулирует обработку через `/health/load-simulate?work_ms=2000` (≈ Wiki+LLM).

---

## Точки мониторинга

| URL | Что смотреть |
|-----|--------------|
| `/health` | Жив ли процесс |
| `/analytics/summary` | Метрики, routing, **логи (снимок)** |
| `/analytics/events` | Сырые события (JSON) |
| `/analytics/logs` | In-memory логи (JSON) |
| `/stats` | Агрегаты за период |

### Дашборд

- **Обновить** — новый снимок метрик и логов (автообновления нет)
- Логи: последние N строк из памяти процесса (после рестарта — пусто до активности)
- Таблицы маршрутизации: все 150 систем + keywords

---

## Логи

Буфер in-memory (~500 строк), логгеры `app.*` и `uvicorn`.

Типичные строки:

```text
Pachca webhook: type=message event=new user=... chat=...
Pachca thread follow-up ok: interaction=... clarification=False sources=1
Analytics dashboard snapshot: 42 log lines
```

При проблемах:

1. Воспроизвести в Pachca
2. Нажать **Обновить** на дашборде
3. Искать `unknown_thread`, `answer_error`, `ticket_create_error`

---

## Данные на диске (не в git)

| Путь | Содержимое |
|------|------------|
| `data/conversations.json` | Контекст тредов |
| `data/support_ai.db` | SQLite аналитика |
| `data/feedback.json` | Feedback memory (👍/👎) |
| `data/pachca_poller_state.json` | Состояние poller |

При рестарте сервера `conversations.json` сохраняется — follow-up в треде работает, если файл на месте.

---

## Доработка в бою

### Keywords (система SD)

Файл: `app/knowledge/tracker_routing.py` → `SYSTEM_KEYWORDS`

Порядок важен: более специфичные keywords выше.

### Приоритеты Wiki

Файл: `app/knowledge/priorities.py`

### Категории обращений

Файл: `app/knowledge/tracker_routing.py` → `CATEGORY_CONTEXT_RULES`, `resolve_category()`

### Матрица очередей

Файл: `app/knowledge/system_sd_matrix.py`

---

## Деплой

1. Pull `main`
2. Перезапуск uvicorn / контейнера
3. Проверка `/health`
4. Тестовое сообщение в Pachca + «Обновить» на дашборде

---

## Runbook (кратко)

| Симптом | Действие |
|---------|----------|
| Бот не отвечает | `/health`, логи, poller/webhook |
| Нет ответа после уточнения ОС | лог `unknown_thread`, проверить thread_chat_id fix |
| Неверная очередь | keywords + локация на дашборде |
| LLM долго думает | 10–20 сек норма; на сообщении таймер `agent-thinking` |
| Сервер «умер» | `./scripts/start_server.sh` или `python scripts/watchdog.py` |
| Кнопки не работают | только автор, один клик; `action_taken` в store |

---

## Скрипты

```bash
./scripts/start_server.sh          # uvicorn + watchdog
./scripts/status.sh
python scripts/scenario_smoke.py   # smoke топ-сценариев (без LLM)
python scripts/load_test.py        # нагрузка
python scripts/watchdog.py         # только watchdog (редко нужен)
python scripts/test_pachca.py      # отправка в Pachca
python scripts/test_pachca_webhook.py
python scripts/inspect_tracker_fields.py
```
