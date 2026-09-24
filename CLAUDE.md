# Фото-отзывы WB

Адаптивный веб-сайт для учёта заказов на фото товаров. Kanban-доска, CRUD заказов, загрузка файлов в Cloudinary, экспорт CSV.

## Поля заказа

- `code` — код заказа (неуникальный, необязательный)
- `title` — название (обязательное)
- `description` — описание
- `price` — стоимость товара
- `workPrice` — стоимость работы
- `paid` — оплачено/неоплачено
- `manager` — менеджер (Даша / Аня / Алина)
- `startDate` — дата начала заказа (указывается вручную, отображается зелёным в карточке)
- `deadline` — срок выполнения
- `status` — 0 = В работе, 1 = Готово

## Структура

Монорепозиторий с npm workspaces:
- `client/` — React 18 + Vite + Tailwind CSS
- `server/` — Node.js + Express + Prisma + PostgreSQL

## Команды

```bash
npm install              # установка зависимостей (все workspaces)
npm run dev              # запуск dev-серверов (клиент + сервер)
npm run build            # сборка клиента для продакшена

# Сервер отдельно
cd server
npx prisma migrate dev   # миграция БД
npx prisma db push       # пуш схемы без миграции
npx prisma generate      # генерация клиента
```

## Переменные окружения

### server/.env
- `NODE_ENV` — `production` на VPS
- `HOST` — `127.0.0.1` за nginx, `0.0.0.0` локально
- `PORT` — порт сервера (по умолчанию 3001)
- `DATABASE_URL` — PostgreSQL connection string (локальный postgres на VPS)
- `AUTH_LOGIN` / `AUTH_PASSWORD` — логин и пароль для авторизации
- `JWT_SECRET` — секрет для подписи JWT
- `CORS_ORIGIN` — прод-домен (например `https://example.com`), используется только при `NODE_ENV=production`
- `CLOUDINARY_CLOUD_NAME` / `CLOUDINARY_API_KEY` / `CLOUDINARY_API_SECRET`

### client
- `VITE_API_BASE_URL` — не требуется при деплое на тот же домен (фронт и API на одном хосте, axios идёт по относительному `/api`)

## Стек

- React 18, Vite, Tailwind CSS, Zustand, React Hook Form + Zod, @dnd-kit, axios, react-hot-toast
- Express, Prisma, PostgreSQL, Cloudinary, jsonwebtoken, csv-stringify

## Авторизация

JWT на базе env-переменных. POST /api/login проверяет credentials, возвращает токен на 7 дней. Все остальные /api/* эндпоинты защищены middleware.

## Деплой

Основной хостинг — **Vercel** (статика + `/api` serverless через `api/index.js` + `vercel.json`) + **Neon** (PostgreSQL, pooled connection). Файлы — Cloudinary. Runbook миграции/отката: [`deploy/VERCEL.md`](deploy/VERCEL.md).

VPS (nginx + PM2 + локальный Postgres) — запасной вариант, инструкция [`deploy/INSTALL.md`](deploy/INSTALL.md). Откат — DNS обратно на IP VPS. Локально `npm run dev` работает как раньше (Express раздаёт статику сам, на Vercel — платформа).

**DNS:** A-запись домена на Vercel (`76.76.21.21`). Cloudflare **не используется как proxy** (DNS-only, серый облако), так как Cloudflare proxy严重限速 для российских VPS.

Артефакты деплоя:
- `vercel.json` — сборка клиента, SPA fallback, реврайт `/api` → `api/index.js`, регион fra1
- `deploy/VERCEL.md` — runbook миграции с VPS и отката
- `deploy/nginx.conf`, `deploy/ecosystem.config.cjs`, `deploy/deploy.sh` — fallback-VPS (nginx → `127.0.0.1:3001`, PM2)
