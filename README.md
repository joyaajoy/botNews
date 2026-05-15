Kimi: # ContentFusion Bot — План реализации (Todo)

## 1. Инфраструктура и окружение
- [ ] Создать репозиторий и структуру монорепозитория:
  - [ ] `apps/bot/` — GramIO бот
  - [ ] `apps/api/` — Backend для MiniApp
  - [ ] `apps/workers/` — Воркеры BullMQ
  - [ ] `apps/miniapp/` — React/Vue MiniApp
  - [ ] `packages/db/` — Prisma-схема и миграции
  - [ ] `packages/shared/` — Types, utils, константы
- [ ] Настроить `docker-compose.yml`:
  - [ ] Сервис `postgres:15-alpine` с volume `pgdata`
  - [ ] Сервис `redis:7-alpine` с volume `redisdata`
  - [ ] Сервисы для бота, API, воркеров (parse + publish)
- [ ] Создать `.env` шаблон:
  - [ ] `BOT_TOKEN`
  - [ ] `ADMIN_CHAT_ID`
  - [ ] `TARGET_CHANNEL_ID`
  - [ ] `DATABASE_URL`
  - [ ] `REDIS_URL`
  - [ ] `OPENAI_API_KEY` (или аналог для NLP)
  - [ ] `MINIAPP_URL`

## 2. База данных (PostgreSQL + Prisma)
- [ ] Инициализировать Prisma в `packages/db/`
- [ ] Описать схему `sources`:
  - [ ] Поля: `id`, `type` (enum: telegram_channel | rss | website), `name`, `url`, `telegram_channel_id`, `is_active`, `check_interval_minutes`, `created_at`
- [ ] Описать схему `source_tags`:
  - [ ] Поля: `id`, `source_id`, `tag`, `filter_mode` (whitelist | blacklist | required), `weight`
- [ ] Описать схему `raw_items`:
  - [ ] Поля: `id`, `source_id`, `external_id`, `title`, `content`, `url`, `media_urls` (массив), `published_at`, `fetched_at`, `hash` (unique)
- [ ] Описать схему `unified_posts`:
  - [ ] Поля: `id`, `raw_item_id`, `status` (pending | approved | rejected | published | scheduled), `formatted_text`, `media_group` (JSONB), `target_channel_id`, `scheduled_at`, `published_at`, `telegram_message_id`, `engagement_score`
- [ ] Описать схему `publish_logs`:
  - [ ] Поля: `id`, `post_id`, `action`, `error`, `created_at`
- [ ] Сгенерировать и применить первую миграцию: `prisma migrate dev`
- [ ] Создать seed-скрипт для тестовых источников

## 3. Redis и очереди (BullMQ)
- [ ] Установить `bullmq` во всех нужных пакетах
- [ ] Создать конфиг подключения к Redis (shared)
- [ ] Инициализировать очередь `parse-content`:
  - [ ] Настроить retry policy (3 попытки, backoff exponential)
- [ ] Инициализировать очередь `publish-content`:
  - [ ] Настроить rate limiting под лимиты Telegram
- [ ] Создать `QueueEvents` для мониторинга failed jobs

## 4. GramIO бот (`apps/bot/`)
- [ ] Инициализировать проект через `npm create gramio@latest`
- [ ] Настроить `.command("start")`:
  - [ ] Приветственное сообщение с `format` (bold/italic)
  - [ ] Inline кнопка: "📊 Открыть Dashboard" → `web_app`
  - [ ] Inline кнопка: "⚙️ Настройки ленты" → `web_app`
- [ ] Настроить `.command("addsource")` (базовая CLI-команда для админов):
  - [ ] Парсинг аргументов: `/addsource <type> <url>`
  - [ ] Валидация URL
  - [ ] Сохранение в БД через Prisma
- [ ] Обработчик `.on("channel_post")`:
  - [ ] Получение `channelId` из `ctx.channelPost.chat.id`
  - [ ] Проверка: есть ли канал в таблице `sources` и активен ли он
  - [ ] Формирование SHA256-хеша: `channelId + content + date`
  - [ ] Проверка дубликата по `hash` в `raw_items`
  - [ ] Если уникально — добавить job в `parseQueue` с payload:
    - [ ] `sourceId`, `externalId`, `title`, `content`, `media` (file_ids), `url`, `hash`, `publishedAt`
- [ ] Обработчик `.on("message")` для WebApp данных:
  - [ ] Проверка `ctx.message.web_app_data?.data`
  - [ ] Парсинг JSON
  - [ ] Action `approve_post`:
    - [ ] Добавить job в `publishQueue` (`immediate: true`)
    - [ ] Ответить пользователю: "✅ Пост отправлен в очередь публикации"
  - [ ] Action `schedule_post`:
    - [ ] Обновить `unified_posts`: `status = scheduled`, `scheduled_at`
    - [ ] Ответить: "📅 Пост запланирован"
  - [ ] Action `reject_post`:
    - [ ] Обновить `unified_posts`: `status = rejected`
    - [ ] Ответить: "❌ Пост отклонен"
- [ ] Настроить `.onStart()`:
  - [ ] Логирование `info.username`
  - [ ] Запуск cron-воркеров RSS и Web (вызов функций из workers)

## 5. Воркер обработки (`apps/workers/processor.ts`)
- [ ] Создать `Worker("parse-content", handler)`:
  - [ ] Получить `sourceId`, `title`, `content`, `media`, `url`, `hash` из job.data
- [ ] Шаг 1 — сохранить raw item:
  - [ ] `db.rawItems.create({ sourceId, title, content, url, mediaUrls: media, hash })`
- [ ] Шаг 2 — NLP извлечение тегов:
  - [ ] Интеграция с OpenAI / compromise.js / локальной моделью
  - [ ] Функция `extractTags(content)` → массив строк
- [ ] Шаг 3 — фильтрация по тегам источника:
  - [ ] Загрузить `sourceTags` для данного `sourceId`
  - [ ] Найти пересечение `matchedTags`
  - [ ] Если `sourceTags` заданы, но `matchedTags` пусты — отбросить (return)
- [ ] Шаг 4 — семантическая дедупликация:
  - [ ] Функция `findSimilarPosts(content)` (поиск по векторам или полнотекстовый)
  - [ ] Если `similarity > 0.92` — отбросить как дубль
- [ ] Шаг 5 — унификация формата:
  - [ ] Функция `formatUnifiedPost({ title, content, sourceUrl, tags, sourceName })`
  - [ ] Шаблон: `bold("📰 " + title)` + `italic(summary)` + теги + ссылка на источник + имя источника
  - [ ] Обрезка контента до 400 символов с `...`
- [ ] Шаг 6 — AI-оценка engagement:
  - [ ] Функция `predictEngagement(title, content)` → score 0..1
- [ ] Шаг 7 — создание `unified_posts`:
  - [ ] `status: "pending"`
  - [ ] `formattedText`, `mediaGroup`, `engagementScore`
- [ ] Шаг 8 — уведомление админа:
  - [ ] `bot.api.sendMessage(ADMIN_CHAT_ID, ...)` с краткой сводкой
  - [ ] Inline кнопка: "📋 Модерация" → `web_app` с параметром `?id=${post.id}`

## 6. Воркер публикации (`apps/workers/publisher.ts`)
- [ ] Создать `Worker("publish-content", handler)`:
  - [ ] Получить `postId`, `immediate` из job.data
  - [ ] Загрузить пост из `unified_posts` по ID
  - [ ] Проверить статус (должен быть `pending` или `immediate === true`)
- [ ] Логика публикации:
  - [ ] `targetChannel = process.env.TARGET_CHANNEL_ID`
  - [ ] Если `mediaGroup` не пустой:
    - [ ] Сформировать массив для `sendMediaGroup` (type: photo, media: file_id, caption на первом элементе)
    - [ ] Вызвать `bot.api.sendMediaGroup(targetChannel, mediaGroup)`
    - [ ] Сохранить массив `message_ids`
  - [ ] Если медиа нет:
    - [ ] Вызвать `bot.api.sendMessage(targetChannel, formattedText, { parse_mode: "HTML" })`
    - [ ] Сохранить `message_id`
- [ ] Обновить запись:
  - [ ] `status: "published"`
  - [ ] `publishedAt: new Date()`
  - [ ] `telegramMessageId: messageIds[0]`
- [ ] Логирование: `console.log("📤 Published post #${postId}")`

## 7. RSS и Web парсеры (Cron-воркеры)
- [ ] RSS Worker:
  - [ ] Библиотека `rss-parser`
  - [ ] Загрузить все активные источники `type = 'rss'`
  - [ ] Для каждого: `parser.parseURL(source.url)`
  - [ ] Для каждого item: проверка `guid`/`link` на дубликат
  - [ ] Добавить в `parseQueue` аналогично Telegram-источникам
- [ ] Web Worker:
  - [ ] Библиотека `cheerio` (статичные) или `puppeteer` (SPA/JS-сайты)
  - [ ] Загрузить активные источники `type = 'website'`
  - [ ] Для каждого: fetch HTML → парсинг заголовков, ссылок, контента
  - [ ] Формирование `hash` и проверка дубликата
  - [ ] Добавить в `parseQueue`
- [ ] Настроить интервал проверки по `check_interval_minutes` из `sources`

## 8. Backend API для MiniApp (`apps/api/`)
- [ ] Инициализировать Fastify
- [ ] Middleware `preHandler`:
  - [ ] Чтение `x-telegram-init-data` из headers
  - [ ] Валидация через `validateTelegramData(initData, BOT_TOKEN)`
  - [ ] Извлечение `user` из initData → `req.user`
- [ ] Эндпоинт `GET /api/stats`:
  - [ ] `pending` — count `unified_posts` where `status = pending`
  - [ ] `todayPublished` — count where `status = published` и `published_at >= startOfDay`
  - [ ] `activeSources` — count `sources` where `is_active = true`
  - [ ] `duplicatesBlocked` — count `raw_items` where `isDuplicate = true` за месяц
  - [ ] `chartData` — агрегация публикаций по дням (7 дней)
  - [ ] `pendingPosts` — top 10 `pending` с `include: { rawItem: { include: { source: true } } }`
- [ ] Эндпоинт `POST /api/sources`:
  - [ ] Body: `type`, `name`, `url`, `tags[]`, `interval`
  - [ ] Валидация URL и типа
  - [ ] `db.sources.create(...)`
  - [ ] Для каждого тега: `db.sourceTags.create({ sourceId, tag, filter_mode: "required" })`
  - [ ] Вернуть `{ id, status: "created" }`
- [ ] Эндпоинт `GET /api/sources`:
  - [ ] Список всех источников с тегами
- [ ] Эндпоинт `PATCH /api/sources/:id`:
  - [ ] Обновление `is_active`, `interval`, тегов
- [ ] Эндпоинт `DELETE /api/sources/:id`:
  - [ ] Soft-delete или каскадное удаление тегов
- [ ] Эндпоинт `GET /api/queue`:
  - [ ] Пагинация pending постов
- [ ] Эндпоинт `POST /api/queue/:id/approve`:
  - [ ] Добавить в `publishQueue`
- [ ] Эндпоинт `POST /api/queue/:id/schedule`:
  - [ ] Body: `scheduledAt`
- [ ] Эндпоинт `POST /api/queue/:id/reject`:
  - [ ] Обновить статус
- [ ] Запуск: `app.listen({ port: 3000 })`

## 9. MiniApp Frontend (`apps/miniapp/`)
- [ ] Инициализировать React + Vite (или Next.js)
- [ ] Установить `recharts` для графиков
- [ ] Настроить Telegram WebApp SDK:
  - [ ] `tg.ready()`, `tg.expand()`, `tg.BackButton.hide()`
  - [ ] Чтение `tg.initData` для авторизации
  - [ ] `tg.sendData()` для отправки действий в бот
- [ ] Страница Dashboard (`/dashboard`):
  - [ ] KPI Cards (2x2 grid): В очереди, Сегодня, Источники, Дедупы
  - [ ] `LineChart` (публикации за 7 дней)
  - [ ] Список "На модерации" (top 10)
  - [ ] Кнопки: ✅ Опубликовать / ❌ Отклонить (через `tg.sendData`)
- [ ] Страница Sources (`/settings` или `/sources`):
  - [ ] Список источников с toggle `is_active`
  - [ ] Форма добавления: тип, URL, название, теги, интервал
  - [ ] Удаление источника
- [ ] Страница Analytics (опционально):
  - [ ] Графики по тегам
  - [ ] Графики по источникам (какой дает больше контента)
  - [ ] Engagement score distribution
- [ ] Настроить роутинг (React Router) с учетом Telegram WebView
- [ ] Стилизация под Telegram темы:
  - [ ] Использовать CSS-переменные: `var(--tg-theme-bg-color)`, `var(--tg-theme-text-color)` и т.д.

## 10. NLP и AI-компоненты
- [ ] Выбрать провайдера: OpenAI GPT-4o-mini / Claude Haiku / локальная модель
- [ ] Реализовать `extractTags(text)`:
  - [ ] Промпт: "Извлеки 3-7 релевантных тегов из текста на русском/английском"
  - [ ] Парсинг ответа → массив строк
- [ ] Реализовать `predictEngagement(title, content)`:
  - [ ] Промпт: "Оцени вероятность интереса аудитории к посту (0-1)"
  - [ ] Или эвристика: длина, наличие ссылок, эмодзи, ключевые слова
- [ ] Реализовать семантический поиск (дедупликация):
  - [ ] Вариант A: OpenAI embeddings + cosine similarity в PostgreSQL (`pgvector`)
  - [ ] Вариант B: Полнотекстовый поиск через `to_tsvector` (быстрее, проще)

## 11. Деплой и DevOps
- [ ] Написать `Dockerfile` для каждого сервиса (bot, api, workers)
- [ ] Настроить `docker-compose.prod.yml`
- [ ] Настроить CI/CD (GitHub Actions):
  - [ ] Lint + TypeScript check
  - [ ] Сборка образов
  - [ ] Деплой на VPS / Railway / Fly.io
- [ ] Настроить мониторинг:
  - [ ] Логи (или просто `docker logs`)
  - [ ] Метрики BullMQ (failed jobs count)
  - [ ] Алертинг в Telegram при падении воркеров

## 12. Тестирование
- [ ] Unit-тесты для `formatUnifiedPost`
- [ ] Unit-тесты для `extractTags` (mock NLP)
- [ ] Unit-тесты для дедупликации (hash + similarity)
- [ ] Интеграционный тест: добавление RSS → парсинг → появление в очереди
- [ ] E2E: approve через MiniApp → публикация в тестовый канал
- [ ] Проверка rate limits Telegram (не более 20 сообщений/мин в один чат)

## 13. Документация и финализация
- [ ] Написать README:
  - [ ] Архитектура (диаграмма)
  - [ ] Инструкция по запуску `docker-compose up`
  - [ ] Описание переменных окружения
  - [ ] Как добавить новый источник
- [ ] Создать CHANGELOG
- [ ] Добавить лицензию (MIT)
- [ ] Финальный деплой MVP в продакшен
