## Техническая основа (GramIO + MiniApp)

### GramIO — ключевые паттерны

```typescript
import { Bot, format, bold } from "gramio";

// 1. Managed Business Bot
const businessBot = new Bot(process.env.BUSINESS_TOKEN)
  .on("business_message", async (ctx) => {
    // Ответ от имени владельца бизнеса
    await ctx.send(format`Привет, ${bold(ctx.from.first_name)}!`);
  })
  .on("business_connection", (ctx) => {
    console.log("New business connection:", ctx.businessConnection.id);
  });

// 2. Bot-to-Bot Communication
const agentBot = new Bot(process.env.AGENT_TOKEN)
  .on("message", async (ctx) => {
    // Проверяем, пришло ли сообщение от другого бота
    if (ctx.from?.is_bot && ctx.from.id !== ctx.me.id) {
      const payload = parseBotProtocol(ctx.text); // кастомный протокол
      await processTask(payload);
      // Ответ обратно через тот же Bot-to-Bot канал
      await ctx.send(formatBotResponse({ status: "done", result: ... }));
    }
  });

// 3. MiniApp Launch
bot.command("dashboard", (ctx) => {
  return ctx.send("Откройте панель управления:", {
    reply_markup: {
      inline_keyboard: [[
        { text: "📊 Открыть MiniApp", web_app: { url: "https://myapp.com/dashboard" } }
      ]]
    }
  });
});

bot.start();
```

### MiniApp — интеграция

```javascript
// Внутри MiniApp (React/Vue)
const tg = window.Telegram.WebApp;
tg.ready();
tg.expand();

// Получение initData для авторизации
const initData = tg.initDataUnsafe;
const businessConnectionId = initData.start_param; // или из backend

// Отправка данных обратно в бот
tg.sendData(JSON.stringify({ action: "approve_order", id: 123 }));
```

---

## Сводная таблица концепций

| # | Название | Business Mode | Bot-to-Bot | MiniApp-роль |
|---|----------|---------------|------------|--------------|
| 1 | Customer Support Swarm | ✅ Координатор | ✅ Агенты | Аналитика CS, takeover |
| 2 | Content Moderation | ❌ | ✅ Pipeline | Модераторский дашборд |
| 3 | E-commerce Fulfillment | ✅ Магазин | ✅ Склад/доставка | Kanban заказов, карта |
| 4 | Multi-Language Concierge | ✅ Бизнес | ✅ Переводчики | Глоссарий, side-by-side |
| 5 | Appointment Orchestrator | ✅ Сервис | ✅ Календарь/CRM | Drag-and-drop календарь |
| 6 | DevOps CI/CD Swarm | ❌ | ✅ Pipeline | Pipeline viz, метрики |
| 7 | Healthcare Triage | ✅ Клиника | ✅ Специалисты | Triage-очередь, аналитика |
| 8 | Cross-Border Trade | ✅ Трейдер | ✅ Таможня/FX | Карта маршрутов, P&L |
| 9 | Educational Collective | ✅ Репетитор | ✅ Предметники | Прогресс, gamification |
| 10 | Real Estate Swarm | ✅ Риелтор | ✅ Показы/договоры | Карта объектов, CRM |

---

**Ключевое преимущество архитектуры:** Bot-to-Bot позволяет масштабировать каждый агент независимо (разные сервера, разные языки), а Business Mode сохраняет человеческое лицо бренда в коммуникации. MiniApp замыкает цикл, давая владельцу полный контроль и визуализацию.
User: бот который собирает в единую ленту публикации в других каналах, rss и сайтах по заданной теме, тегам и формирует унифицированные посты в одну ленту или канал
Kimi: # Концепция: **ContentFusion Bot** — Агрегатор контента с унифицированной лентой

## Общая архитектура

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│  Telegram       │     │  RSS-ленты      │     │  Веб-сайты      │
│  Каналы-источники│     │  (XML/Atom)     │     │  (Scraping/API) │
└────────┬────────┘     └────────┬────────┘     └────────┬────────┘
         │                       │                       │
         └───────────────────────┼───────────────────────┘
                                 ▼
                    ┌─────────────────────┐
                    │   Parser Workers    │  ← Cheerio/Puppeteer/rss-parser
                    │   (Bull Queue)      │
                    └──────────┬──────────┘
                               ▼
                    ┌─────────────────────┐
                    │   AI/Keyword Filter │  ← Теги, NLP, дедупликация
                    │   & Deduplicator    │
                    └──────────┬──────────┘
                               ▼
                    ┌─────────────────────┐
                    │   Unify Formatter   │  ← Единый шаблон оформления
                    │   (Markdown/HTML)   │
                    └──────────┬──────────┘
                               ▼
                    ┌─────────────────────┐
                    │   Publisher Bot     │  ← GramIO
                    │   (Target Channel)  │
                    └──────────┬──────────┘
                               ▼
                    ┌─────────────────────┐
                    │   MiniApp Dashboard │  ← Управление, очередь, аналитика
                    │   (React/Vue)       │
                    └─────────────────────┘
```

---

## Технический стек

| Компонент | Технология |
|-----------|------------|
| Bot Framework | **GramIO** (TypeScript) |
| Queue / Jobs | **BullMQ** (Redis) |
| Database | **PostgreSQL** + **Redis** (cache/dedup) |
| Parsing | `rss-parser`, `cheerio`, `puppeteer` (для JS-сайтов) |
| NLP/Filter | `compromise` или внешний API (OpenAI/Anthropic) |
| MiniApp Frontend | React + Telegram WebApp SDK |
| Backend for MiniApp | Fastify/Express API |

---

## Схема данных (PostgreSQL)

```sql
-- Источники контента
CREATE TABLE sources (
    id SERIAL PRIMARY KEY,
    type VARCHAR(20) CHECK (type IN ('telegram_channel', 'rss', 'website')),
    name VARCHAR(255),
    url VARCHAR(500),           -- Ссылка на RSS или сайт
    telegram_channel_id BIGINT, -- Для TG-каналов
    is_active BOOLEAN DEFAULT true,
    check_interval_minutes INT DEFAULT 30,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Теги/фильтры для источника
CREATE TABLE source_tags (
    id SERIAL PRIMARY KEY,
    source_id INT REFERENCES sources(id) ON DELETE CASCADE,
    tag VARCHAR(100),           -- Например: "AI", "crypto", "python"
    filter_mode VARCHAR(20) CHECK (filter_mode IN ('whitelist', 'blacklist', 'required')),
    weight FLOAT DEFAULT 1.0    -- Вес для ранжирования
);

-- Сырые записи (raw content)
CREATE TABLE raw_items (
    id SERIAL PRIMARY KEY,
    source_id INT REFERENCES sources(id),
    external_id VARCHAR(500), -- ID из TG или guid из RSS
    title TEXT,
    content TEXT,
    url TEXT,
    media_urls TEXT[],         -- Массив ссылок на медиа
    published_at TIMESTAMP,
    fetched_at TIMESTAMP DEFAULT NOW(),
    hash VARCHAR(64) UNIQUE    -- SHA256 для дедупликации
);

-- Унифицированные посты (очередь публикации)
CREATE TABLE unified_posts (
    id SERIAL PRIMARY KEY,
    raw_item_id INT REFERENCES raw_items(id),
    status VARCHAR(20) CHECK (status IN ('pending', 'approved', 'rejected', 'published', 'scheduled')),
    formatted_text TEXT,       -- Итоговый текст поста
    media_group JSONB,         -- Медиа для отправки
    target_channel_id BIGINT,
    scheduled_at TIMESTAMP,
    published_at TIMESTAMP,
    telegram_message_id BIGINT,
    engagement_score FLOAT     -- Предиктивный скор интереса
);

-- Логи публикаций
CREATE TABLE publish_logs (
    id SERIAL PRIMARY KEY,
    post_id INT REFERENCES unified_posts(id),
    action VARCHAR(50),
    error TEXT,
    created_at TIMESTAMP DEFAULT NOW()
);
```

---

## Реализация на GramIO

### 1. Основной бот-агрегатор (`aggregator-bot.ts`)

```typescript
import { Bot, format, bold, italic, link } from "gramio";
import { BullMQ } from "bullmq";
import { createHash } from "crypto";

// --- Конфигурация ---
const bot = new Bot(process.env.BOT_TOKEN as string);
const redisConnection = { host: "localhost", port: 6379 };

// --- Очереди ---
const parseQueue = new BullMQ("parse-content", { connection: redisConnection });
const publishQueue = new BullMQ("publish-content", { connection: redisConnection });

// --- Команды ---

// /start — приветствие + кнопка MiniApp
bot.command("start", (ctx) => {
  return ctx.send(
    format`${bold("🧠 ContentFusion Bot")}\n\nАгрегатор контента из каналов, RSS и сайтов.\n\nОткройте панель управления:`,
    {
      reply_markup: {
        inline_keyboard: [
          [
            {
              text: "📊 Открыть Dashboard",
              web_app: { url: process.env.MINIAPP_URL + "/dashboard" },
            },
          ],
          [
            {
              text: "⚙️ Настройки ленты",
              web_app: { url: process.env.MINIAPP_URL + "/settings" },
            },
          ],
        ],
      },
    }
  );
});

// /addsource — добавление источника (упрощенно, в проде через MiniApp)
bot.command("addsource", async (ctx) => {
  const [type, url] = ctx.args;
  // Сохранение в БД...
  await ctx.send(format`✅ Источник ${code(url)} добавлен. Тип: ${type}`);
});

// --- Обработка входящих из Telegram-каналов ---

// Бот должен быть админом в канале-источнике для получения обновлений
bot.on("channel_post", async (ctx) => {
  const channelId = ctx.channelPost.chat.id;
  
  // Проверяем, есть ли такой источник в БД
  const source = await db.sources.findByChannelId(channelId);
  if (!source || !source.is_active) return;

  // Формируем хеш для дедупликации
  const content = ctx.channelPost.text || ctx.channelPost.caption || "";
  const hash = createHash("sha256")
    .update(`${channelId}:${content}:${ctx.channelPost.forward_date || ctx.channelPost.date}`)
    .digest("hex");

  // Проверяем дубликат
  const exists = await db.rawItems.findByHash(hash);
  if (exists) return;

  // Добавляем в очередь обработки
  await parseQueue.add("process-telegram", {
    sourceId: source.id,
    externalId: `${channelId}_${ctx.channelPost.message_id}`,
    title: content.slice(0, 100),
    content,
    media: ctx.channelPost.photo?.map((p) => p.file_id) || [],
    url: `https://t.me/${ctx.channelPost.chat.username}/${ctx.channelPost.message_id}`,
    hash,
    publishedAt: new Date(ctx.channelPost.date * 1000),
  });

  console.log(`📥 New item queued from channel ${channelId}`);
});

// --- Обработка callback от MiniApp ---
bot.on("message", async (ctx) => {
  // WebApp отправляет данные через sendData → приходит как обычное сообщение с web_app_data
  if (ctx.message?.web_app_data?.data) {
    const data = JSON.parse(ctx.message.web_app_data.data);
    
    switch (data.action) {
      case "approve_post":
        await publishQueue.add("publish", { postId: data.postId, immediate: true });
        await ctx.send("✅ Пост отправлен в очередь публикации");
        break;
        
      case "schedule_post":
        await db.unifiedPosts.update(data.postId, {
          status: "scheduled",
          scheduledAt: new Date(data.scheduledAt),
        });
        await ctx.send(format`📅 Пост запланирован на ${data.scheduledAt}`);
        break;
        
      case "reject_post":
        await db.unifiedPosts.update(data.postId, { status: "rejected" });
        await ctx.send("❌ Пост отклонен");
        break;
    }
  }
});

// --- Запуск ---
bot.onStart(({ info }) => {
  console.log(`🤖 ContentFusion Bot @${info.username} started`);
  
  // Запускаем cron-воркеры для RSS и Web
  startRssWorker();
  startWebWorker();
});

bot.start();
```

### 2. Воркер обработки (`workers/processor.ts`)

```typescript
import { Worker } from "bullmq";
import { OpenAI } from "openai"; // или локальный NLP

const nlp = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });

// --- Worker: Обработка сырых записей ---
const processWorker = new Worker(
  "parse-content",
  async (job) => {
    const { sourceId, title, content, media, url, hash } = job.data;

    // 1. Сохраняем raw item
    const rawItem = await db.rawItems.create({
      sourceId,
      title,
      content,
      url,
      mediaUrls: media,
      hash,
    });

    // 2. Извлекаем теги / темы (NLP)
    const tags = await extractTags(content);
    const sourceTags = await db.sourceTags.findBySource(sourceId);
    
    // 3. Фильтрация: проверяем соответствие тегам источника
    const matchedTags = tags.filter((t) => sourceTags.some((st) => st.tag === t));
    if (sourceTags.length > 0 && matchedTags.length === 0) {
      console.log(`⛔ Filtered out: no matching tags for ${url}`);
      return;
    }

    // 4. Дедупликация по семантике (поиск похожих)
    const similar = await findSimilarPosts(content);
    if (similar && similar.similarity > 0.92) {
      console.log(`♻️ Duplicate detected (similarity: ${similar.similarity})`);
      return;
    }

    // 5. Форматирование унифицированного поста
    const formatted = await formatUnifiedPost({
      title,
      content,
      sourceUrl: url,
      tags: matchedTags,
      sourceName: (await db.sources.findById(sourceId)).name,
    });

    // 6. Создаем unified post в статусе pending
    const post = await db.unifiedPosts.create({
      rawItemId: rawItem.id,
      status: "pending",
      formattedText: formatted.text,
      mediaGroup: media,
      engagementScore: formatted.score,
    });

    // 7. Уведомление админа через бот (опционально)
    await bot.api.sendMessage(
      process.env.ADMIN_CHAT_ID,
      format`🆕 Новый пост в очереди\n\n${bold(title)}\nТеги: ${tags.join(", ")}\n\nОткройте дашборд для модерации.`,
      {
        reply_markup: {
          inline_keyboard: [
            [{ text: "📋 Модерация", web_app: { url: `${process.env.MINIAPP_URL}/moderate?id=${post.id}` } }],
          ],
        },
      }
    );
  },
  { connection: redisConnection }
);

// --- Форматтер ---
async function formatUnifiedPost(data: {
  title: string;
  content: string;
  sourceUrl: string;
  tags: string[];
  sourceName: string;
}) {
  // Унифицированный шаблон
  const text = format`
${bold("📰 " + data.title)}

${italic(data.content.slice(0, 400) + (data.content.length > 400 ? "..." : ""))}

🏷 ${data.tags.map((t) => `#${t}`).join(" ")}
🔗 ${link("Источник", data.sourceUrl)}
🗞 ${italic(data.sourceName)}
`;

  // AI-оценка интереса (0-1)
  const score = await predictEngagement(data.title, data.content);

  return { text, score };
}
```

### 3. Воркер публикации (`workers/publisher.ts`)

```typescript
import { Worker } from "bullmq";

const publishWorker = new Worker(
  "publish-content",
  async (job) => {
    const { postId, immediate } = job.data;
    const post = await db.unifiedPosts.findById(postId);
    if (!post || post.status !== "pending" && !immediate) return;

    const targetChannel = process.env.TARGET_CHANNEL_ID;

    // Публикация в канал
    let messageIds: number[] = [];

    if (post.mediaGroup && post.mediaGroup.length > 0) {
      // Альбом
      const mediaGroup = post.mediaGroup.map((fileId, idx) => ({
        type: "photo" as const,
        media: fileId,
        caption: idx === 0 ? post.formattedText : undefined,
        parse_mode: "HTML" as const,
      }));
      
      const result = await bot.api.sendMediaGroup(targetChannel, mediaGroup);
      messageIds = result.map((m) => m.message_id);
    } else {
      // Текст
      const result = await bot.api.sendMessage(targetChannel, post.formattedText, {
        parse_mode: "HTML",
        disable_web_page_preview: false,
      });
      messageIds = [result.message_id];
    }

    // Обновляем статус
    await db.unifiedPosts.update(postId, {
      status: "published",
      publishedAt: new Date(),
      telegramMessageId: messageIds[0],
    });

    console.log(`📤 Published post #${postId} to channel`);
  },
  { connection: redisConnection }
);
```

---

## MiniApp — Панель управления

### Структура экранов

| Экран | Назначение |
|-------|------------|
| **Dashboard** | Общая статистика, график публикаций, источники |
| **Sources** | Управление источниками (добавить RSS/TG/сайт, включить/выключить) |
| **Queue** | Очередь на модерацию: approve/reject/schedule |
| **Analytics** | Аналитика по источникам, тегам, engagement |
| **Settings** | Шаблоны оформления, теги, интервалы проверки |

### React-компонент Dashboard (упрощенно)

```tsx
// src/pages/Dashboard.tsx
import { useEffect, useState } from "react";
import { LineChart, Line, XAxis, YAxis, Tooltip } from "recharts";

export default function Dashboard() {
  const tg = window.Telegram.WebApp;
  const [stats, setStats] = useState(null);

  useEffect(() => {
    tg.ready();
    tg.expand();
    tg.BackButton.hide();
    
    // Загрузка данных с backend
    fetch(`${process.env.REACT_APP_API}/api/stats`, {
      headers: { "X-Telegram-Init-Data": tg.initData },
    })
      .then((r) => r.json())
      .then(setStats);
  }, []);

  const handleApprove = (postId: number) => {
    tg.sendData(JSON.stringify({ action: "approve_post", postId }));
    tg.showAlert("✅ Отправлено на публикацию");
  };

  return (
    <div className="p-4 bg-[var(--tg-theme-bg-color)] text-[var(--tg-theme-text-color)]">
      <h1 className="text-xl font-bold mb-4">📊 ContentFusion</h1>
      
      {/* KPI Cards */}
      <div className="grid grid-cols-2 gap-3 mb-6">
        <StatCard title="В очереди" value={stats?.pending || 0} color="orange" />
        <StatCard title="Сегодня" value={stats?.todayPublished || 0} color="green" />
        <StatCard title="Источники" value={stats?.activeSources || 0} color="blue" />
        <StatCard title="Дедупов" value={stats?.duplicatesBlocked || 0} color="red" />
      </div>

      {/* График публикаций */}
      <div className="bg-[var(--tg-theme-secondary-bg-color)] rounded-xl p-3 mb-6">
        <h3 className="text-sm font-semibold mb-2">Публикации за 7 дней</h3>
        <LineChart width={320} height={150} data={stats?.chartData}>
          <XAxis dataKey="date" tick={{ fontSize: 10 }} />
          <YAxis tick={{ fontSize: 10 }} />
          <Tooltip />
          <Line type="monotone" dataKey="count" stroke="#2481cc" strokeWidth={2} />
        </LineChart>
      </div>

      {/* Очередь модерации */}
      <h3 className="font-semibold mb-3">🕐 На модерации</h3>
      <div className="space-y-3">
        {stats?.pendingPosts?.map((post) => (
          <div key={post.id} className="bg-[var(--tg-theme-secondary-bg-color)] rounded-lg p-3">
            <div className="text-sm font-medium mb-1">{post.title}</div>
            <div className="text-xs text-gray-500 mb-2">
              {post.sourceName} • {post.tags.join(", ")}
            </div>
            <div className="flex gap-2">
              <button
                onClick={() => handleApprove(post.id)}
                className="flex-1 bg-green-500 text-white text-sm py-2 rounded-lg"
              >
                ✅ Опубликовать
              </button>
              <button
                onClick={() => tg.sendData(JSON.stringify({ action: "reject_post", postId: post.id }))}
                className="flex-1 bg-red-500 text-white text-sm py-2 rounded-lg"
              >
                ❌ Отклонить
              </button>
            </div>
          </div>
        ))}
      </div>
    </div>
  );
}
```

### Backend API для MiniApp (`api/server.ts`)

```typescript
import Fastify from "fastify";
import { validateTelegramData } from "./telegram-auth";

const app = Fastify();

// Middleware: проверка initData от Telegram
app.addHook("preHandler", async (req, reply) => {
  const initData = req.headers["x-telegram-init-data"] as string;
  if (!initData || !validateTelegramData(initData, process.env.BOT_TOKEN)) {
    return reply.status(401).send({ error: "Unauthorized" });
  }
  req.user = parseInitData(initData).user;
});

// Статистика для дашборда
app.get("/api/stats", async (req) => {
  const userId = req.user.id;
  
  const [pending, today, sources, duplicates, chart, pendingPosts] = await Promise.all([
    db.unifiedPosts.count({ status: "pending" }),
    db.unifiedPosts.count({ status: "published", publishedAt: { gte: startOfDay() } }),
    db.sources.count({ is_active: true }),
    db.rawItems.count({ isDuplicate: true, createdAt: { gte: startOfMonth() } }),
    getChartData(7),
    db.unifiedPosts.findMany({
      where: { status: "pending" },
      take: 10,
      orderBy: { createdAt: "desc" },
      include: { rawItem: { include: { source: true } } },
    }),
  ]);

  return {
    pending,
    todayPublished: today,
    activeSources: sources,
    duplicatesBlocked: duplicates,
    chartData: chart,
    pendingPosts: pendingPosts.map((p) => ({
      id: p.id,
      title: p.rawItem.title,
      sourceName: p.rawItem.source.name,
      tags: p.rawItem.tags,
      engagementScore: p.engagementScore,
    })),
  };
});

// CRUD источников
app.post("/api/sources", async (req) => {
  const { type, name, url, tags, interval } = req.body;
  const source = await db.sources.create({ type, name, url, check_interval_minutes: interval });
  
  // Добавляем теги
  for (const tag of tags) {
    await db.sourceTags.create({ sourceId: source.id, tag, filter_mode: "required" });
  }
  
  return { id: source.id, status: "created" };
});

app.listen({ port: 3000 });
```

---

## Docker Compose для развертывания

```yaml
version: "3.8"

services:
  bot:
    build: ./bot
    env_file: .env
    depends_on:
      - postgres
      - redis
    command: npm run start:bot

  worker-parse:
    build: ./bot
    env_file: .env
    depends_on:
      - postgres
      - redis
    command: npm run start:worker:parse

  worker-publish:
    build: ./bot
    env_file: .env
    depends_on:
      - postgres
      - redis
    command: npm run start:worker:publish

  api:
    build: ./miniapp-api
    env_file: .env
    ports:
      - "3000:3000"
    depends_on:
      - postgres

  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: contentfusion
      POSTGRES_USER: fusion
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - pgdata:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    volumes:
      - redisdata:/data

volumes:
  pgdata:
  redisdata:
```

---

## Ключевые фичи системы

| Фича | Реализация |
|------|------------|
| **Дедупликация** | SHA256-хеш + семантический поиск по эмбеддингам |
| **Smart Filter** | Теги на источник + NLP-извлечение тем из текста |
| **Унификация** | Единый Markdown-шаблон с автоматическим summary |
| **Модерация** | MiniApp с очередью approve/reject/schedule |
| **Аналитика** | Графики по источникам, тегам, времени публикаций |
| **Масштабирование** | BullMQ-воркеры, независимое масштабирование парсеров |

---

## Возможные расширения

1. **AI-рерайт** — перед публикацией прогонять через LLM для уникализации текста
2. **Cross-posting** — публикация сразу в несколько каналов с адаптацией формата
3. **Автопостинг по расписанию** — "лучшее время" на основе аналитики engagement
4. **Комментарии/обсуждения** — интеграция с группой для обсуждения постов
5. **Поиск по архиву** — векторный поиск по всему собранному контенту

Эта архитектура дает полный контроль над контент-пайплайном через MiniApp, сохраняя при этом автоматизацию сбора и фильтрации.
User: можешь свой ответ оформить в виде Todo в формате markdown?
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
