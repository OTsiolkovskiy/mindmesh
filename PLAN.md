# MindMesh — план розробки

Персональний AI Knowledge OS: нотатки + семантичний пошук + RAG-чат, який відповідає на основі власних нотаток користувача.

## Ціль кінцевого продукту

Повноцінний, задеплоєний full-stack застосунок, де користувач:
1. Реєструється / логіниться (email+password, Google/GitHub OAuth)
2. Створює, редагує, тегує нотатки (markdown)
3. Прикріплює файли/зображення до нотаток
4. Шукає нотатки за змістом (семантичний пошук через векторні embeddings), а не тільки за ключовими словами
5. Спілкується з AI-чатом, який під час відповіді підтягує релевантні нотатки користувача як контекст (RAG) і відповідає з посиланням, з яких нотаток взята інформація
6. Бачить синхронізацію в реальному часі, якщо відкрито кілька вкладок
7. Довіряє системі — дані регулярно бекапляться автоматично

## Стек (фіксований, не змінювати без причини)

**Фронтенд**
- Next.js 15, App Router, TypeScript (strict mode)
- Tailwind CSS + shadcn/ui
- TanStack Query — клієнтський кеш/стан для даних, що не йдуть через Server Components
- Vercel AI SDK (v5) — стрімінг відповідей AI в UI

**Бекенд/дані**
- Supabase: Postgres, Auth, Storage, Realtime, Edge Functions (Deno)
- pgvector — розширення Postgres для векторного пошуку
- Row Level Security (RLS) на кожній таблиці з user_id — без винятків

**AI-шар**
- Claude API (Anthropic) — генерація відповідей чату
- Embeddings-модель (Voyage AI або OpenAI text-embedding) — векторизація нотаток для RAG

**Інфраструктура (безплатний tier)**
- Vercel — хостинг фронтенду, автодеплой з GitHub
- Supabase Free — БД, auth, storage
- GitHub Actions — CI, щоденні бекапи БД (pg_dump → Storage bucket або артефакт)
- Sentry (free tier) — моніторинг помилок

## Схема бази даних (базова, буде розширюватись)

```sql
-- Нотатки
notes (
  id uuid primary key default gen_random_uuid(),
  user_id uuid references auth.users not null,
  title text not null,
  content text not null,        -- markdown
  created_at timestamptz default now(),
  updated_at timestamptz default now()
)

-- Векторні представлення чанків нотаток (для RAG)
note_embeddings (
  id uuid primary key default gen_random_uuid(),
  note_id uuid references notes on delete cascade,
  chunk_text text not null,
  embedding vector(1536),        -- розмірність залежить від обраної embeddings-моделі
  created_at timestamptz default now()
)

-- Чат-сесії
chat_sessions (
  id uuid primary key default gen_random_uuid(),
  user_id uuid references auth.users not null,
  title text,
  created_at timestamptz default now()
)

-- Повідомлення чату
chat_messages (
  id uuid primary key default gen_random_uuid(),
  session_id uuid references chat_sessions on delete cascade,
  role text check (role in ('user', 'assistant')),
  content text not null,
  cited_note_ids uuid[],          -- які нотатки використані як контекст
  created_at timestamptz default now()
)

-- Теги
tags (
  id uuid primary key default gen_random_uuid(),
  user_id uuid references auth.users not null,
  name text not null
)

note_tags (
  note_id uuid references notes on delete cascade,
  tag_id uuid references tags on delete cascade,
  primary key (note_id, tag_id)
)
```

Кожна таблиця з `user_id` (напряму або через join) отримує RLS-політику виду:
```sql
create policy "Users manage own notes" on notes
  for all using (auth.uid() = user_id);
```

## RAG-пайплайн (як це працює технічно)

1. Користувач створює/редагує нотатку
2. Edge Function (тригер on insert/update) розбиває текст на чанки, викликає embeddings API, зберігає вектори в `note_embeddings`
3. Коли користувач пише в чат:
   - Питання векторизується
   - `pgvector` cosine similarity search знаходить топ-N релевантних чанків користувача
   - Ці чанки підмішуються в system prompt Claude API як контекст
   - Відповідь стрімиться в UI через Vercel AI SDK
   - Зберігається, які note_id були використані (для показу "джерел" у відповіді)

## Етапи розробки (виконувати послідовно, кожен — окрема сесія Claude Code)

### Етап 0 — Готово
- [x] Ініціалізація Next.js 15 (App Router, TypeScript, Tailwind)
- [x] Git-репозиторій на GitHub

### Етап 1 — Supabase Auth
- [ ] Підключити @supabase/supabase-js, @supabase/ssr
- [ ] Ключі API — тільки нова система Supabase: publishable (`sb_publishable_...`) і secret (`sb_secret_...`); legacy `anon` / `service_role` не використовуємо
- [ ] `.env.local`: `NEXT_PUBLIC_SUPABASE_URL`, `NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY`, `SUPABASE_SECRET_KEY` (+ `.env.example` без значень у репо)
- [ ] Supabase-клієнти в `lib/supabase/`: browser і server (через `@supabase/ssr`) на publishable key; admin-клієнт на secret key — тільки server-side
- [ ] Email/password реєстрація й логін
- [ ] OAuth (Google, GitHub)
- [ ] Захищені роути (middleware)

### Етап 2 — Схема БД
- [ ] Створити міграції для всіх таблиць вище
- [ ] Увімкнути pgvector extension
- [ ] RLS-політики на кожній таблиці
- [ ] Перевірити через Supabase Dashboard, що політики працюють

### Етап 3 — CRUD нотаток
- [ ] Список нотаток, створення, редагування, видалення
- [ ] Markdown-редактор (TipTap або BlockNote)
- [ ] Server Actions для мутацій

### Етап 4 — Векторизація (RAG pipeline, частина 1)
- [ ] Edge Function: чанкінг тексту + виклик embeddings API
- [ ] Тригер на insert/update нотатки
- [ ] Збереження в note_embeddings

### Етап 5 — Семантичний пошук
- [ ] UI пошуку
- [ ] Векторизація запиту + cosine similarity search через pgvector
- [ ] Показ релевантних нотаток

### Етап 6 — RAG-чат (частина 2)
- [ ] UI чату (список сесій, повідомлення)
- [ ] Retrieval релевантних чанків під запит
- [ ] Виклик Claude API зі стрімінгом (Vercel AI SDK)
- [ ] Показ "джерел" — які нотатки використані у відповіді

### Етап 7 — Файли та зображення
- [ ] Supabase Storage bucket для вкладень
- [ ] Прикріплення до нотаток, попередній перегляд

### Етап 8 — Realtime
- [ ] Supabase Realtime channels — синхронізація нотаток між вкладками

### Етап 9 — Автотегування (опційно)
- [ ] Edge Function викликає AI при створенні нотатки → генерує теги й короткий опис

### Етап 10 — Деплой
- [ ] Vercel: імпорт репозиторію, env-змінні
- [ ] Автодеплой з main branch
- [ ] Перевірка production build

### Етап 11 — Бекапи та моніторинг
- [ ] GitHub Action: щоденний `supabase db dump` → приватний Storage/артефакт, ротація 7/30 днів
- [ ] Sentry для моніторингу помилок фронтенду й Edge Functions

## Прийняті рішення (не переглядати без вагомої причини)

- **pgvector замість зовнішньої vector DB** (Pinecone тощо) — все в одному Postgres, простіше для pet-проекту, безплатно в межах Supabase free tier
- **Server Actions замість окремих API routes** — менше boilerplate, ближче до Next.js 15 ідіом
- **RLS обов'язковий на кожній таблиці з user_id** — безпека на рівні БД, а не тільки на рівні коду
- **Vercel AI SDK для стрімінгу** — стандарт для AI-чатів у Next.js, вбудована підтримка Claude
- **Нові API-ключі Supabase (publishable/secret) замість legacy anon/service_role** — legacy JWT-ключі виводяться з підтримки; нові ключі можна ротувати незалежно, а secret key Supabase відхиляє при виклику з браузера
- **Бекапи через GitHub Actions**, не через платні Supabase Pro бекапи — щоб лишатись у безплатному tier

## Що НЕ входить в MVP (можна додати пізніше)

- Спільні простори / шеринг нотаток між користувачами
- Публічні сторінки нотаток
- Офлайн-режим / PWA
- Мобільний застосунок
