# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

# MindMesh

Персональний AI Knowledge OS: нотатки з семантичним пошуком і RAG-чатом на основі власних нотаток користувача. Повний план та етапи розробки — дивись PLAN.md в корені репозиторію; цей файл описує тільки стек і конвенції.

## Стек
- Next.js 15, App Router, TypeScript (strict mode)
- Tailwind CSS v4 (через `@tailwindcss/postcss`) + shadcn/ui
- TanStack Query для клієнтського стану/кешу
- Vercel AI SDK (v5) для стрімінгу AI-відповідей
- Supabase: Postgres + pgvector, Auth, Storage, Realtime, Edge Functions (Deno)
- Claude API (Anthropic) — генерація відповідей чату
- Embeddings-модель (Voyage AI / OpenAI) — векторизація нотаток для RAG

Наразі встановлені лише Next.js, React, Tailwind і ESLint — решту стеку додаємо по мірі проходження етапів із PLAN.md.

## Команди
- `npm run dev` — dev-сервер (Turbopack) на http://localhost:3000
- `npm run build` — production build (Turbopack), запускати перед деплоєм
- `npm run start` — запуск production build
- `npm run lint` — ESLint
- `npx supabase db push` — застосувати міграції (коли з'явиться `supabase/`)
- `npm run test` — тести (test runner ще не налаштований)

## Структура проекту
Один Next.js застосунок у корені репозиторію, без `src/` — роути лежать прямо в `app/`.

```
mindmesh/
├── app/                   # роути, layouts, route-level UI (page.tsx, layout.tsx, ...)
├── components/            # спільні перевикористовувані UI-компоненти
├── lib/                   # framework-agnostic хелпери, доступ до даних, бізнес-логіка
├── types/                 # спільні TypeScript-типи/інтерфейси
├── public/                # статичні ассети
├── supabase/              # (заплановано)
│   ├── migrations/        # SQL-міграції, версійовані
│   ├── functions/         # Edge Functions (Deno)
│   └── seed.sql
├── CLAUDE.md
└── PLAN.md
```

Аліас `@/*` вказує на корінь проекту (див. `tsconfig.json`), тому імпорти між папками — `@/components/...`, `@/lib/...`, `@/types/...`, а не відносні шляхи.

`components/`, `lib/` і `types/` поки що порожні (лише `.gitkeep`) — наповнюються по мірі появи коду.

## Конвенції
- Server Actions замість API routes, де це можливо
- Усі зміни схеми БД — тільки через нову міграцію в `supabase/migrations`, ніколи прямі зміни в Dashboard без відповідної міграції в репо
- RLS-політика обов'язкова на кожній таблиці, що містить `user_id` — без винятків
- Компоненти: PascalCase, файли `.tsx`
- Векторний пошук — тільки через pgvector, не додавати зовнішні vector DB
- Секрети (API keys, service role key) — тільки через `.env.local`, ніколи не хардкодити і не комітити

## Що НЕ робити
- Не чіпати `supabase/migrations` вручну без створення нового migration-файлу
- Не хардкодити API-ключі — тільки через змінні середовища
- Не додавати нові великі залежності (vector DB, ORM, стейт-менеджери) без явного обговорення — стек зафіксований в PLAN.md

## Поточний статус
Дивись розділ "Етапи розробки" в PLAN.md — там позначено, що вже зроблено, що в роботі, що далі.
