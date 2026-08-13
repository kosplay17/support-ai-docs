# Support AI — документация

Материалы для обсуждения архитектуры, внедрения и стандартизации AI-решений в Lamoda.

**Текущая версия продукта:** v0.7  
**Публичное описание:** [https://github.com/kosplay17/support-ai-docs](https://github.com/kosplay17/support-ai-docs)

> Исходный код Support AI — **закрытый репозиторий** (доступ у команды Support AI / IT).

---

## Содержание

| Документ | Для чего |
|----------|----------|
| [architecture.md](architecture.md) | Схема решения, потоки данных, слои |
| [components.md](components.md) | Из чего состоит система (модули и ответственность) |
| [integrations.md](integrations.md) | Внешние системы: Pachca, Wiki, Tracker, LLM |
| [roadmap.md](roadmap.md) | Версии v0.1–v2.0, North Star, критерии v1.0 |
| [operations.md](operations.md) | Запуск, мониторинг, логи, доработка в бою |
| [standardization.md](standardization.md) | Подход к стандартизации AI-решений в компании |

### Визуальные схемы

| Файл | Формат | Содержание |
|------|--------|------------|
| [diagrams/support-ai-architecture.drawio](diagrams/support-ai-architecture.drawio) | draw.io | 3 страницы: архитектура, слои, поток обращения |
| [diagrams/support-ai-architecture.svg](diagrams/support-ai-architecture.svg) | SVG | Общая схема + 8 слоёв (открыть в браузере) |
| [diagrams/support-ai-architecture.png](diagrams/support-ai-architecture.png) | PNG | Экспорт для слайдов |

Открыть draw.io: [diagrams.net](https://app.diagrams.net) → File → Open → выбрать `.drawio`

---

## Одним абзацем

Support AI — **корпоративный AI Service Desk**: пользователь пишет в Pachca → бот ищет инструкции в Яндекс Wiki → отвечает в треде → при необходимости создаёт тикет в Яндекс Tracker с автоматической маршрутизацией (система SD, локация, очередь, категория).

---

## Для встречи / async

Рекомендуемый порядок презентации:

1. **standardization.md** — зачем единый подход и чем Support AI отличается от «ещё одного чат-бота»
2. **architecture.md** — общая схема (1–2 слайда)
3. **components.md** — слои и границы ответственности
4. **roadmap.md** — что уже сделано (v0.1–v0.7), v1.0 и путь к ИИ-инженеру (v1.1–v2.0)
