# Roadmap Support AI

Хронология версий и планы — для синхронизации с другими AI-инициативами в компании.

---

## North Star — ИИ-инженер технической поддержки

Целевой образ продукта **после v1.0** — не «чат-бот с Wiki», а **агент L1/L1.5**, который:

| Способность | Сейчас (v0.7) | Целевая версия |
|-------------|---------------|----------------|
| Ответы по внутренней Wiki | ✅ | — |
| Поиск похожих инцидентов в Tracker | — | v1.2 |
| Анализ логов (корреляция с симптомом) | — | v1.4 |
| Интеграция с Tracker (создание, статус, комментарии) | ✅ частично | v1.2 |
| Запуск PowerShell-скриптов диагностики | — | v1.3 |
| Проверки AD, VPN, MDM, статус устройств | — (intent VPN в диалоге) | v1.3 |
| Вероятная причина неисправности | — | v1.4 |
| Ежедневная аналитика SLA и CSAT | ✅ базовый дашборд | v1.5 |
| Problem Management-отчёты и CSI-инициативы | — | v1.6 |

**v1.0** — закрываем **стабильный L1 на Wiki + Tracker** локально.  
Всё из таблицы выше — **после v1.0**, итерациями с human-in-the-loop и audit trail.

---

## Статус версий

| Версия | Статус | Суть |
|--------|--------|------|
| **v0.1** | ✅ | FastAPI + LLM |
| **v0.2** | ✅ | Wiki search |
| **v0.2.1** | ✅ | Собственный ranking |
| **v0.2.2** | ✅ | Ручные приоритеты статей |
| **v0.2.3** | ✅ | Feedback memory |
| **v0.3** | ✅ | Pachca: webhook, треды |
| **v0.4** | ✅ | Кнопки, защита кликов, аналитика |
| **v0.5** | ✅ | Tracker: тикеты, автор из Pachca |
| **v0.6** | ✅ | Матрица SD, keywords, routing, дашборд |
| **v0.7** | ✅ | Обучение на feedback, burst, multi-intent |
| **v1.0** | ✅ | Production Service Desk (локально) |
| **v1.1** | ✅ | Channel policies (офис / тех. каналы) |
| **v1.2** | 🚧 | Tracker intelligence + knowledge layer |
| **v1.3** | 📋 | Диагностические tools (PS / AD / VPN / MDM) |
| **v1.4** | 📋 | Логи и root cause |
| **v1.5** | 📋 | SLA / CSAT ops analytics |
| **v1.6** | 📋 | Problem Management и CSI |
| **v2.0** | 💡 | Hosted production, multi-channel, agent platform |

---

## Детализация по версиям

### v0.1 — LLM-ядро
- FastAPI, endpoint `/llm/ask`
- Подключение корпоративного LLM

### v0.2 — Wiki
- API Яндекс Wiki, пространство `cf/`
- Передача инструкций в LLM

### v0.2.1–v0.2.3 — Качество поиска
- Scoring, приоритеты, feedback memory
- Снижение «не той инструкции»

### v0.3 — Pachca
- Webhook + poller
- Треды, история диалога

### v0.4 — Обратная связь
- 👍 / 👎 / 🎫
- SQLite-аналитика
- Защита кнопок (автор, один клик)

### v0.5 — Tracker
- Создание тикетов из Pachca
- Автор = пользователь Pachca
- Description с диалогом и ссылкой на тред

### v0.6 — Маршрутизация SD
- 150 систем SD → очередь
- 149 systems с keywords
- Категория по контексту диалога
- Диагностика оборудования (Wi‑Fi, кнопки)
- HTML-дашборд + logs snapshot
- Follow-up в треде (thread.message_chat_id)

### v0.7 — Обучение и устойчивость диалога

- 👍 → `feedback.json` → буст `correct_slug` в ranking
- 👎 → `feedback.json` → штраф `wrong_slug` в ranking
- `GET /learning/report` — топ без инструкции + keyword suggestions
- Метка `no_instruction` в analytics для отчётов
- Секция «Обучение» на `/analytics/summary`
- Burst в канале (5 сек): склейка сообщений одного автора → один тред
- Coalesce в активный тред (1 мин) + раннее сохранение сессии
- Multi-intent, VPN troubleshoot, статус задачи из нескольких очередей (`TRACKER_STATUS_QUEUES`)
- Маршрутизация «Нет в списке (другой)» по локации (SUPBYK и др.)

### v1.0 — Production *(цель, локальная эксплуатация)*

**Хостинг на сервере отложен** — бот крутится локально; критерии v1.0 — качество и стабильность L1-процесса.

| Критерий | Описание |
|----------|----------|
| Стабильность | `./scripts/start_server.sh` (uvicorn + watchdog), `./scripts/status.sh` |
| Покрытие | Топ-сценарии SD + `python scripts/scenario_smoke.py` |
| Deflection | 👍 / ответы — KPI на дашборде (ориентир ≥30%) |
| Wiki coverage | Доля с инструкцией, `no_instruction` ≤25% |
| Routing accuracy | <10% ручных правок SD-полей |
| Gaps | `/learning/report` — приоритеты для Wiki |
| Burst / coalesce | Быстрые серии сообщений → один тред, без дублей |
| Принятие SD | Support officially использует бота как L1 |

**Не входит в v1.0:** автодиагностика инфраструктуры, анализ логов, PM-отчёты — см. v1.2+.

---

## После v1.0 — путь к ИИ-инженеру

### v1.1 — Channel policies *(сделано)*

Расширение канала Pachca без смены core.

- `chat_id → mode` (`full_support` / `office_hybrid`)
- Классификация: technical vs office vs шум
- Офисный канал: IT-флоу для тех. проблем; консультации только при hit в Wiki
- Офисный контент в Wiki
- Аналитика: `channel_mode`, `silenced`

Что уже внедрено:

- Отдельный Office Hotline flow для канала `7710693`
- Structured Wiki FAQ для Office Hotline (`Route`, `Canonical`, `Answer`, `Aliases`, `Examples`, `Notes`)
- Dedicated analytics page: `/analytics/office-hotline`
- FAQ-only ответы при confidence ≥ 0.95
- Разделение technical / office FAQ / silent
- Спецкейс `IT Storage` с видео-маршрутом и fallback на Wiki attachment
- Read-only режим для калибровки и последующее включение live-ответов
- Auto-close follow-up сессии после сообщения пользователя вида “решилось / помогло / всё работает”

### v1.2 — Tracker intelligence + knowledge layer *(MVP в работе)*

Первый шаг от «ответ по Wiki» к «контекст из истории компании».

| Задача | Описание |
|--------|----------|
| Похожие инциденты | Поиск закрытых задач Tracker по симптому / системе / keywords; top-N в prompt |
| Обогащение тикета | При создании — ссылки на похожие задачи, known workaround |
| Комментарии в тред | Смена статуса / комментарий инженера → уведомление автора в Pachca |
| RAG / embeddings | Qdrant (или аналог) для Wiki + выдержки из resolved tickets при росте объёма |
| Admin UI | Редактирование keywords / priorities / channel map без деплоя |
| A/B ranking | Сравнение стратегий retrieval (rule-based vs hybrid) |

Что уже есть в MVP:

- `POST /v3/issues/_search` для поиска resolved задач Tracker по текстовым keywords
- В description создаваемого тикета добавляется секция `Похожие решенные задачи Tracker`
- Источник keywords: `system + location + question`
- Окно поиска: последние ~90 дней

**Критерий готовности:** в ≥20% обращений бот цитирует релевантный прошлый инцидент; false-positive rate контролируется вручную.

### v1.3 — Диагностические tools

Controlled agent layer: **явный whitelist** команд, audit log, подтверждение пользователя на опасные действия.

| Tool | Назначение |
|------|------------|
| PowerShell runner | Запуск утверждённых скриптов (VPN, диск, сеть, принтер) в sandbox / jump host |
| AD lookup | Группы, блокировка, срок пароля, последний logon |
| VPN | Статус сессии, профиль, тип клиента |
| MDM / Intune | Compliance, enrollment, последняя синхронизация |
| Device inventory | Hostname → владелец, ОС, локация (из CMDB / AD) |

- Orchestration: `dialogue_service` решает *нужен ли* tool; выполнение — отдельный `diagnostics_service`
- Результаты — structured JSON в тред + опционально в поле Tracker
- Без произвольного shell: только catalog scripts v1

**Критерий готовности:** 3–5 частых сценариев (VPN, диск Z:, Wi‑Fi, пароль AD) закрываются диагностикой без эскалации.

### v1.4 — Логи и root cause

| Задача | Описание |
|--------|----------|
| Источники логов | VPN gateway, MDM, AD sync, Wi‑Fi controller — по whitelist |
| Корреляция | Симптом из Pachca + время + user/device → релевантные строки |
| Root cause hint | LLM формулирует *вероятную* причину с цитатами из логов (grounded) |
| Эскалация | Автопредложение очереди / приоритета при совпадении с known pattern |

**Принцип:** LLM не «угадывает» — только интерпретирует найденные факты из логов и Wiki.

### v1.5 — SLA / CSAT ops analytics

Ежедневная операционная аналитика для руководителя SD.

| Метрика | Источник |
|---------|----------|
| Deflection rate | SQLite analytics (уже есть) |
| CSAT proxy | 👍/👎 ratio, ticket-after-answer |
| SLA | Tracker: time-to-first-response, time-to-resolve по очередям |
| Routing quality | % ручных правок полей, top misrouted systems |
| Wiki gaps | `/learning/report` trends |

- Scheduled report в Pachca / email (канал SD leads)
- Drill-down на дашборде: очередь, система, локация, период
- Базeline и алерты при деградации KPI

### v1.6 — Problem Management и CSI

| Задача | Описание |
|--------|----------|
| Clustering | Группировка повторяющихся обращений за период (по системе + симптому) |
| Problem draft | Черновик Problem ticket: описание, затронутые системы, частота, примеры задач |
| CSI initiatives | Предложения по улучшению Wiki / процесса / автоматизации на основе топ-clusters |
| Monthly PM report | Авто-сборка для Problem Management review |

Human approval обязателен перед созданием Problem / изменением процесса.

### v2.0 — Platform *(vision)*

| Направление | Описание |
|-------------|----------|
| Hosted production | Деплой на сервер, HA, секреты, мониторинг |
| Multi-channel | Email, портал SD, тот же core + channel adapters |
| Multi-tenant | Шаблон для других доменов (HR, Finance) |
| Tool registry | Общий каталог diagnostics tools для других AI-агентов компании |
| Governance | Единый audit, quota LLM, политики данных |

---

## Принцип развития

```text
v0.x:  LLM → Wiki → Ranking → Pachca → Feedback → Tracker → Categorization → Learning
v1.0:  Стабильный L1 (локально, KPI, калибровка)
v1.x:  Knowledge++ → Tools → Logs → Ops analytics → Problem Management
v2.0:  Platform (hosting, channels, tenant)
```

Каждый этап — **отдельный законченный слой**. Новые AI-проекты в компании могут повторять порядок v0.x, не начиная с «сразу agent с PowerShell».

**Guardrails для v1.3+:**

- Whitelist tools, no arbitrary code execution
- Audit log каждого tool call (interaction_id, user, args, result)
- Пользователь видит, что проверял бот, до эскалации
- Sensitive data — маскирование в логах и analytics

---

## Калибровка в бою

Для v0.6+ **не используются** автотесты маршрутизации — keywords и категории дорабатываются по реальным обращениям. Это осознанный trade-off: быстрая итерация важнее CI-тестов на 150 систем.

Для v1.3+ (tools) — обязательны **contract tests** на catalog scripts и sandbox перед prod.

---

## Связь с корпоративной стандартизацией

См. [standardization.md](standardization.md) — как roadmap Support AI ложится на общие «рельсы» AI-разработки в Lamoda.  
Support AI после v1.0 — **reference implementation** для agent-with-tools в домене ITSM.
