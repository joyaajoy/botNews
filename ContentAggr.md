**User:**  
бот который собирает в единую ленту публикации в других каналах, rss и сайтах по заданной теме, тегам и формирует унифицированные посты в одну ленту или канал  
---

# Концепция: **ContentFusion Bot** — Агрегатор контента с унифицированной лентой

## Общая архитектура

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

---

## Технический стек

| Компонент | Технология |
| :---- | :---- |
| Bot Framework | **GramIO** (TypeScript) |
| Queue / Jobs | **BullMQ** (Redis) |
| Database | **PostgreSQL** \+ **Redis** (cache/dedup) |
| Parsing | rss-parser, cheerio, puppeteer (для JS-сайтов) |
| NLP/Filter | compromise или внешний API (OpenAI/Anthropic) |
| MiniApp Frontend | React \+ Telegram WebApp SDK |
| Backend for MiniApp | Fastify/Express API |

---

## Схема данных (PostgreSQL)

*\-- Источники контента*  
**CREATE** **TABLE** sources (  
    **id** SERIAL **PRIMARY** **KEY**,  
    **type** VARCHAR(20) **CHECK** (**type** **IN** ('telegram\_channel', 'rss', 'website')),  
    name VARCHAR(255),  
    url VARCHAR(500),           *\-- Ссылка на RSS или сайт*  
    telegram\_channel\_id BIGINT, *\-- Для TG-каналов*  
    is\_active BOOLEAN **DEFAULT** **true**,  
    check\_interval\_minutes INT **DEFAULT** 30,  
    created\_at TIMESTAMP **DEFAULT** NOW()  
);

*\-- Теги/фильтры для источника*  
**CREATE** **TABLE** source\_tags (  
    **id** SERIAL **PRIMARY** **KEY**,  
    source\_id INT **REFERENCES** sources(**id**) **ON** **DELETE** **CASCADE**,  
    tag VARCHAR(100),           *\-- Например: "AI", "crypto", "python"*  
    filter\_mode VARCHAR(20) **CHECK** (filter\_mode **IN** ('whitelist', 'blacklist', 'required')),  
    weight FLOAT **DEFAULT** 1.0    *\-- Вес для ранжирования*  
);

*\-- Сырые записи (raw content)*  
**CREATE** **TABLE** raw\_items (  
    **id** SERIAL **PRIMARY** **KEY**,  
    source\_id INT **REFERENCES** sources(**id**),  
    external\_id VARCHAR(500), *\-- ID из TG или guid из RSS*  
    title TEXT,  
    content TEXT,  
    url TEXT,  
    media\_urls TEXT\[\],         *\-- Массив ссылок на медиа*  
    published\_at TIMESTAMP,  
    fetched\_at TIMESTAMP **DEFAULT** NOW(),  
    **hash** VARCHAR(64) **UNIQUE**    *\-- SHA256 для дедупликации*  
);

*\-- Унифицированные посты (очередь публикации)*  
**CREATE** **TABLE** unified\_posts (  
    **id** SERIAL **PRIMARY** **KEY**,  
    raw\_item\_id INT **REFERENCES** raw\_items(**id**),  
    status VARCHAR(20) **CHECK** (status **IN** ('pending', 'approved', 'rejected', 'published', 'scheduled')),  
    formatted\_text TEXT,       *\-- Итоговый текст поста*  
    media\_group JSONB,         *\-- Медиа для отправки*  
    target\_channel\_id BIGINT,  
    scheduled\_at TIMESTAMP,  
    published\_at TIMESTAMP,  
    telegram\_message\_id BIGINT,  
    engagement\_score FLOAT     *\-- Предиктивный скор интереса*  
);

*\-- Логи публикаций*  
**CREATE** **TABLE** publish\_logs (  
    **id** SERIAL **PRIMARY** **KEY**,  
    post\_id INT **REFERENCES** unified\_posts(**id**),  
    action VARCHAR(50),  
    error TEXT,  
    created\_at TIMESTAMP **DEFAULT** NOW()  
);

---

## Реализация на GramIO

### 1\. Основной бот-агрегатор (aggregator-bot.ts)

**import** { Bot, format, bold, italic, link } **from** "gramio";  
**import** { BullMQ } **from** "bullmq";  
**import** { createHash } **from** "crypto";

*// \--- Конфигурация \---*  
**const** bot \= **new** Bot(process.env.BOT\_TOKEN **as** string);  
**const** redisConnection \= { host: "localhost", port: 6379 };

*// \--- Очереди \---*  
**const** parseQueue \= **new** BullMQ("parse-content", { connection: redisConnection });  
**const** publishQueue \= **new** BullMQ("publish-content", { connection: redisConnection });

*// \--- Команды \---*

*// /start — приветствие \+ кнопка MiniApp*  
bot.command("start", (ctx) **\=\>** {  
  **return** ctx.send(  
    format\`${bold("🧠 ContentFusion Bot")}\\n\\nАгрегатор контента из каналов, RSS и сайтов.\\n\\nОткройте панель управления:\`,  
    {  
      reply\_markup: {  
        inline\_keyboard: \[  
          \[  
            {  
              text: "📊 Открыть Dashboard",  
              web\_app: { url: process.env.MINIAPP\_URL \+ "/dashboard" },  
            },  
          \],  
          \[  
            {  
              text: "⚙️ Настройки ленты",  
              web\_app: { url: process.env.MINIAPP\_URL \+ "/settings" },  
            },  
          \],  
        \],  
      },  
    }  
  );  
});

*// /addsource — добавление источника (упрощенно, в проде через MiniApp)*  
bot.command("addsource", **async** (ctx) **\=\>** {  
  **const** \[**type**, url\] \= ctx.args;  
  *// Сохранение в БД...*  
  **await** ctx.send(format\`✅ Источник ${code(url)} добавлен. Тип: ${**type**}\`);  
});

*// \--- Обработка входящих из Telegram-каналов \---*

*// Бот должен быть админом в канале-источнике для получения обновлений*  
bot.on("channel\_post", **async** (ctx) **\=\>** {  
  **const** channelId \= ctx.channelPost.chat.id;  
    
  *// Проверяем, есть ли такой источник в БД*  
  **const** source \= **await** db.sources.findByChannelId(channelId);  
  **if** (\!source || \!source.is\_active) **return**;

  *// Формируем хеш для дедупликации*  
  **const** content \= ctx.channelPost.text || ctx.channelPost.caption || "";  
  **const** hash \= createHash("sha256")  
    .update(\`${channelId}:${content}:${ctx.channelPost.forward\_date || ctx.channelPost.date}\`)  
    .digest("hex");

  *// Проверяем дубликат*  
  **const** exists \= **await** db.rawItems.findByHash(hash);  
  **if** (exists) **return**;

  *// Добавляем в очередь обработки*  
  **await** parseQueue.add("process-telegram", {  
    sourceId: source.id,  
    externalId: \`${channelId}\_${ctx.channelPost.message\_id}\`,  
    title: content.slice(0, 100),  
    content,  
    media: ctx.channelPost.photo?.map((p) **\=\>** p.file\_id) || \[\],  
    url: \`https://t.me/${ctx.channelPost.chat.username}/${ctx.channelPost.message\_id}\`,  
    hash,  
    publishedAt: **new** Date(ctx.channelPost.date \* 1000),  
  });

  console.log(\`📥 New item queued from channel ${channelId}\`);  
});

*// \--- Обработка callback от MiniApp \---*  
bot.on("message", **async** (ctx) **\=\>** {  
  *// WebApp отправляет данные через sendData → приходит как обычное сообщение с web\_app\_data*  
  **if** (ctx.message?.web\_app\_data?.data) {  
    **const** data \= JSON.parse(ctx.message.web\_app\_data.data);  
      
    **switch** (data.action) {  
      **case** "approve\_post":  
        **await** publishQueue.add("publish", { postId: data.postId, immediate: **true** });  
        **await** ctx.send("✅ Пост отправлен в очередь публикации");  
        **break**;  
          
      **case** "schedule\_post":  
        **await** db.unifiedPosts.update(data.postId, {  
          status: "scheduled",  
          scheduledAt: **new** Date(data.scheduledAt),  
        });  
        **await** ctx.send(format\`📅 Пост запланирован на ${data.scheduledAt}\`);  
        **break**;  
          
      **case** "reject\_post":  
        **await** db.unifiedPosts.update(data.postId, { status: "rejected" });  
        **await** ctx.send("❌ Пост отклонен");  
        **break**;  
    }  
  }  
});

*// \--- Запуск \---*  
bot.onStart(({ info }) **\=\>** {  
  console.log(\`🤖 ContentFusion Bot @${info.username} started\`);  
    
  *// Запускаем cron-воркеры для RSS и Web*  
  startRssWorker();  
  startWebWorker();  
});

bot.start();

### 2\. Воркер обработки (workers/processor.ts)

**import** { Worker } **from** "bullmq";  
**import** { OpenAI } **from** "openai"; *// или локальный NLP*

**const** nlp \= **new** OpenAI({ apiKey: process.env.OPENAI\_API\_KEY });

*// \--- Worker: Обработка сырых записей \---*  
**const** processWorker \= **new** Worker(  
  "parse-content",  
  **async** (job) **\=\>** {  
    **const** { sourceId, title, content, media, url, hash } \= job.data;

    *// 1\. Сохраняем raw item*  
    **const** rawItem \= **await** db.rawItems.create({  
      sourceId,  
      title,  
      content,  
      url,  
      mediaUrls: media,  
      hash,  
    });

    *// 2\. Извлекаем теги / темы (NLP)*  
    **const** tags \= **await** extractTags(content);  
    **const** sourceTags \= **await** db.sourceTags.findBySource(sourceId);  
      
    *// 3\. Фильтрация: проверяем соответствие тегам источника*  
    **const** matchedTags \= tags.filter((t) **\=\>** sourceTags.some((st) **\=\>** st.tag \=== t));  
    **if** (sourceTags.length \> 0 && matchedTags.length \=== 0) {  
      console.log(\`⛔ Filtered out: no matching tags for ${url}\`);  
      **return**;  
    }

    *// 4\. Дедупликация по семантике (поиск похожих)*  
    **const** similar \= **await** findSimilarPosts(content);  
    **if** (similar && similar.similarity \> 0.92) {  
      console.log(\`♻️ Duplicate detected (similarity: ${similar.similarity})\`);  
      **return**;  
    }

    *// 5\. Форматирование унифицированного поста*  
    **const** formatted \= **await** formatUnifiedPost({  
      title,  
      content,  
      sourceUrl: url,  
      tags: matchedTags,  
      sourceName: (**await** db.sources.findById(sourceId)).name,  
    });

    *// 6\. Создаем unified post в статусе pending*  
    **const** post \= **await** db.unifiedPosts.create({  
      rawItemId: rawItem.id,  
      status: "pending",  
      formattedText: formatted.text,  
      mediaGroup: media,  
      engagementScore: formatted.score,  
    });

    *// 7\. Уведомление админа через бот (опционально)*  
    **await** bot.api.sendMessage(  
      process.env.ADMIN\_CHAT\_ID,  
      format\`🆕 Новый пост в очереди\\n\\n${bold(title)}\\nТеги: ${tags.join(", ")}\\n\\nОткройте дашборд для модерации.\`,  
      {  
        reply\_markup: {  
          inline\_keyboard: \[  
            \[{ text: "📋 Модерация", web\_app: { url: \`${process.env.MINIAPP\_URL}/moderate?id=${post.id}\` } }\],  
          \],  
        },  
      }  
    );  
  },  
  { connection: redisConnection }  
);

*// \--- Форматтер \---*  
**async** **function** formatUnifiedPost(data: {  
  title: string;  
  content: string;  
  sourceUrl: string;  
  tags: string\[\];  
  sourceName: string;  
}) {  
  *// Унифицированный шаблон*  
  **const** text \= format\`  
${bold("📰 " \+ data.title)}

${italic(data.content.slice(0, 400) \+ (data.content.length \> 400 ? "..." : ""))}

🏷 ${data.tags.map((t) **\=\>** \`\#${t}\`).join(" ")}  
🔗 ${link("Источник", data.sourceUrl)}  
🗞 ${italic(data.sourceName)}  
\`;

  *// AI-оценка интереса (0-1)*  
  **const** score \= **await** predictEngagement(data.title, data.content);

  **return** { text, score };  
}

### 3\. Воркер публикации (workers/publisher.ts)

**import** { Worker } **from** "bullmq";

**const** publishWorker \= **new** Worker(  
  "publish-content",  
  **async** (job) **\=\>** {  
    **const** { postId, immediate } \= job.data;  
    **const** post \= **await** db.unifiedPosts.findById(postId);  
    **if** (\!post || post.status \!== "pending" && \!immediate) **return**;

    **const** targetChannel \= process.env.TARGET\_CHANNEL\_ID;

    *// Публикация в канал*  
    **let** messageIds: number\[\] \= \[\];

    **if** (post.mediaGroup && post.mediaGroup.length \> 0) {  
      *// Альбом*  
      **const** mediaGroup \= post.mediaGroup.map((fileId, idx) **\=\>** ({  
        type: "photo" **as** **const**,  
        media: fileId,  
        caption: idx \=== 0 ? post.formattedText : **undefined**,  
        parse\_mode: "HTML" **as** **const**,  
      }));  
        
      **const** result \= **await** bot.api.sendMediaGroup(targetChannel, mediaGroup);  
      messageIds \= result.map((m) **\=\>** m.message\_id);  
    } **else** {  
      *// Текст*  
      **const** result \= **await** bot.api.sendMessage(targetChannel, post.formattedText, {  
        parse\_mode: "HTML",  
        disable\_web\_page\_preview: **false**,  
      });  
      messageIds \= \[result.message\_id\];  
    }

    *// Обновляем статус*  
    **await** db.unifiedPosts.update(postId, {  
      status: "published",  
      publishedAt: **new** Date(),  
      telegramMessageId: messageIds\[0\],  
    });

    console.log(\`📤 Published post \#${postId} to channel\`);  
  },  
  { connection: redisConnection }  
);

---

## MiniApp — Панель управления

### Структура экранов

| Экран | Назначение |
| :---- | :---- |
| **Dashboard** | Общая статистика, график публикаций, источники |
| **Sources** | Управление источниками (добавить RSS/TG/сайт, включить/выключить) |
| **Queue** | Очередь на модерацию: approve/reject/schedule |
| **Analytics** | Аналитика по источникам, тегам, engagement |
| **Settings** | Шаблоны оформления, теги, интервалы проверки |

### React-компонент Dashboard (упрощенно)

// src/pages/Dashboard.tsx  
import { useEffect, useState } from "react";  
import { LineChart, Line, XAxis, YAxis, Tooltip } from "recharts";

export default function Dashboard() {  
  const tg \= window.Telegram.WebApp;  
  const \[stats, setStats\] \= useState(null);

  useEffect(() \=\> {  
    tg.ready();  
    tg.expand();  
    tg.BackButton.hide();  
      
    // Загрузка данных с backend  
    fetch(\`${process.env.REACT\_APP\_API}/api/stats\`, {  
      headers: { "X-Telegram-Init-Data": tg.initData },  
    })  
      .then((r) \=\> r.json())  
      .then(setStats);  
  }, \[\]);

  const handleApprove \= (postId: number) \=\> {  
    tg.sendData(JSON.stringify({ action: "approve\_post", postId }));  
    tg.showAlert("✅ Отправлено на публикацию");  
  };

  return (  
    \<div className="p-4 bg-\[var(--tg-theme-bg-color)\] text-\[var(--tg-theme-text-color)\]"\>  
      \<h1 className="text-xl font-bold mb-4"\>📊 ContentFusion\</h1\>  
        
      {/\* KPI Cards \*/}  
      \<div className="grid grid-cols-2 gap-3 mb-6"\>  
        \<StatCard title="В очереди" value={stats?.pending || 0} color="orange" /\>  
        \<StatCard title="Сегодня" value={stats?.todayPublished || 0} color="green" /\>  
        \<StatCard title="Источники" value={stats?.activeSources || 0} color="blue" /\>  
        \<StatCard title="Дедупов" value={stats?.duplicatesBlocked || 0} color="red" /\>  
      \</div\>

      {/\* График публикаций \*/}  
      \<div className="bg-\[var(--tg-theme-secondary-bg-color)\] rounded-xl p-3 mb-6"\>  
        \<h3 className="text-sm font-semibold mb-2"\>Публикации за 7 дней\</h3\>  
        \<LineChart width={320} height={150} data={stats?.chartData}\>  
          \<XAxis dataKey="date" tick={{ fontSize: 10 }} /\>  
          \<YAxis tick={{ fontSize: 10 }} /\>  
          \<Tooltip /\>  
          \<Line type="monotone" dataKey="count" stroke="\#2481cc" strokeWidth={2} /\>  
        \</LineChart\>  
      \</div\>

      {/\* Очередь модерации \*/}  
      \<h3 className="font-semibold mb-3"\>🕐 На модерации\</h3\>  
      \<div className="space-y-3"\>  
        {stats?.pendingPosts?.map((post) \=\> (  
          \<div key={post.id} className="bg-\[var(--tg-theme-secondary-bg-color)\] rounded-lg p-3"\>  
            \<div className="text-sm font-medium mb-1"\>{post.title}\</div\>  
            \<div className="text-xs text-gray-500 mb-2"\>  
              {post.sourceName} • {post.tags.join(", ")}  
            \</div\>  
            \<div className="flex gap-2"\>  
              \<button  
                onClick={() \=\> handleApprove(post.id)}  
                className="flex-1 bg-green-500 text-white text-sm py-2 rounded-lg"  
              \>  
                ✅ Опубликовать  
              \</button\>  
              \<button  
                onClick={() \=\> tg.sendData(JSON.stringify({ action: "reject\_post", postId: post.id }))}  
                className="flex-1 bg-red-500 text-white text-sm py-2 rounded-lg"  
              \>  
                ❌ Отклонить  
              \</button\>  
            \</div\>  
          \</div\>  
        ))}  
      \</div\>  
    \</div\>  
  );  
}

### Backend API для MiniApp (api/server.ts)

**import** Fastify **from** "fastify";  
**import** { validateTelegramData } **from** "./telegram-auth";

**const** app \= Fastify();

*// Middleware: проверка initData от Telegram*  
app.addHook("preHandler", **async** (req, reply) **\=\>** {  
  **const** initData \= req.headers\["x-telegram-init-data"\] **as** string;  
  **if** (\!initData || \!validateTelegramData(initData, process.env.BOT\_TOKEN)) {  
    **return** reply.status(401).send({ error: "Unauthorized" });  
  }  
  req.user \= parseInitData(initData).user;  
});

*// Статистика для дашборда*  
app.get("/api/stats", **async** (req) **\=\>** {  
  **const** userId \= req.user.id;  
    
  **const** \[pending, today, sources, duplicates, chart, pendingPosts\] \= **await** Promise.all(\[  
    db.unifiedPosts.count({ status: "pending" }),  
    db.unifiedPosts.count({ status: "published", publishedAt: { gte: startOfDay() } }),  
    db.sources.count({ is\_active: **true** }),  
    db.rawItems.count({ isDuplicate: **true**, createdAt: { gte: startOfMonth() } }),  
    getChartData(7),  
    db.unifiedPosts.findMany({  
      where: { status: "pending" },  
      take: 10,  
      orderBy: { createdAt: "desc" },  
      include: { rawItem: { include: { source: **true** } } },  
    }),  
  \]);

  **return** {  
    pending,  
    todayPublished: today,  
    activeSources: sources,  
    duplicatesBlocked: duplicates,  
    chartData: chart,  
    pendingPosts: pendingPosts.map((p) **\=\>** ({  
      id: p.id,  
      title: p.rawItem.title,  
      sourceName: p.rawItem.source.name,  
      tags: p.rawItem.tags,  
      engagementScore: p.engagementScore,  
    })),  
  };  
});

*// CRUD источников*  
app.post("/api/sources", **async** (req) **\=\>** {  
  **const** { **type**, name, url, tags, interval } \= req.body;  
  **const** source \= **await** db.sources.create({ **type**, name, url, check\_interval\_minutes: interval });  
    
  *// Добавляем теги*  
  **for** (**const** tag **of** tags) {  
    **await** db.sourceTags.create({ sourceId: source.id, tag, filter\_mode: "required" });  
  }  
    
  **return** { id: source.id, status: "created" };  
});

app.listen({ port: 3000 });

---

## Docker Compose для развертывания

version**:** "3.8"

services**:**  
  bot**:**  
    build**:** ./bot  
    env\_file**:** .env  
    depends\_on**:**  
      **\-** postgres  
      **\-** redis  
    command**:** npm run start:bot

  worker-parse**:**  
    build**:** ./bot  
    env\_file**:** .env  
    depends\_on**:**  
      **\-** postgres  
      **\-** redis  
    command**:** npm run start:worker:parse

  worker-publish**:**  
    build**:** ./bot  
    env\_file**:** .env  
    depends\_on**:**  
      **\-** postgres  
      **\-** redis  
    command**:** npm run start:worker:publish

  api**:**  
    build**:** ./miniapp-api  
    env\_file**:** .env  
    ports**:**  
      **\-** "3000:3000"  
    depends\_on**:**  
      **\-** postgres

  postgres**:**  
    image**:** postgres:15-alpine  
    environment**:**  
      POSTGRES\_DB**:** contentfusion  
      POSTGRES\_USER**:** fusion  
      POSTGRES\_PASSWORD**:** ${DB\_PASSWORD}  
    volumes**:**  
      **\-** pgdata:/var/lib/postgresql/data

  redis**:**  
    image**:** redis:7-alpine  
    volumes**:**  
      **\-** redisdata:/data

volumes**:**  
  pgdata**:**  
  redisdata**:**

---

## Ключевые фичи системы

| Фича | Реализация |
| :---- | :---- |
| **Дедупликация** | SHA256-хеш \+ семантический поиск по эмбеддингам |
| **Smart Filter** | Теги на источник \+ NLP-извлечение тем из текста |
| **Унификация** | Единый Markdown-шаблон с автоматическим summary |
| **Модерация** | MiniApp с очередью approve/reject/schedule |
| **Аналитика** | Графики по источникам, тегам, времени публикаций |
| **Масштабирование** | BullMQ-воркеры, независимое масштабирование парсеров |

---

## Возможные расширения

1. **AI-рерайт** — перед публикацией прогонять через LLM для уникализации текста

2. **Cross-posting** — публикация сразу в несколько каналов с адаптацией формата

3. **Автопостинг по расписанию** — “лучшее время” на основе аналитики engagement

4. **Комментарии/обсуждения** — интеграция с группой для обсуждения постов

5. **Поиск по архиву** — векторный поиск по всему собранному контенту

Эта архитектура дает полный контроль над контент-пайплайном через MiniApp, сохраняя при этом автоматизацию сбора и фильтрации.