# Два репозитория: код (private) + документация (public)

## Схема

| Репозиторий | Видимость | Содержимое |
|-------------|-----------|------------|
| [support-ai](https://github.com/kosplay17/support-ai) | **Private** | Код бота, скрипты, `.env.example`, внутренний README |
| [support-ai-docs](https://github.com/kosplay17/support-ai-docs) | **Public** | README (описание продукта), `docs/`, диаграммы |

Публичный README и docs синхронизируются скриптом из этого репозитория.

---

## Первичная настройка (один раз)

### 1. Создать public-репозиторий на GitHub

- Имя: `support-ai-docs`
- Visibility: **Public**
- Без README / .gitignore (скрипт создаст содержимое локально)

### 2. Синхронизировать документацию

```bash
./scripts/sync_public_docs.sh
cd ../support-ai-docs
git init
git branch -M main
git remote add origin git@github.com:kosplay17/support-ai-docs.git
git add -A
git commit -m "Initial public documentation"
git push -u origin main
```

### 3. Сделать support-ai приватным

GitHub → **support-ai** → Settings → General → Danger Zone → **Change visibility** → **Private**.

Collaborators: только команда Support AI / IT.

### 4. Проверка

- Без доступа к private repo: открывается `github.com/kosplay17/support-ai-docs` с README и docs.
- `github.com/kosplay17/support-ai` — 404 для посторонних.

---

## Обновление публичной документации

После правок в `docs/` или `README.public.md`:

```bash
./scripts/sync_public_docs.sh
cd ../support-ai-docs
git add -A
git commit -m "Sync docs from support-ai"
git push
```

---

## Файлы-источники

| Файл | Назначение |
|------|------------|
| `README.public.md` | Публичный README (без quick start, env, структуры кода) |
| `README.md` | Внутренний README для разработчиков |
| `docs/` | Документация (копируется в public repo) |
| `scripts/sync_public_docs.sh` | Скрипт синхронизации |
