# Миграция на Vercel + Neon (с откатом на VPS)

Цель: 0 ₽/мес, доступ из РФ. VPS остаётся рабочим до полной проверки — **откат = DNS обратно на IP VPS**.

## 0. Бэкап (на VPS)

```bash
# git-тег уже создан: pre-vercel
sudo -u postgres pg_dump -Fc <dbname> -f ~/pre-vercel-$(date +%F).dump
```

`<dbname>` — имя БД из `server/.env` (DATABASE_URL). Скачать дамп себе на комп и оставить копию на VPS.

## 1. Neon (бесплатная БД)

1. Регистрация на neon.tech — из РФ регистрация заблокирована, нужен VPN **один раз** (сама БД после этого доступна без VPN)
2. Создать проект, регион **eu-central-1** (Франкфурт)
3. В Connection details взять **Pooled connection** (важно для serverless, Prisma без пулинга упрётся в лимиты коннектов)

## 2. Перенос данных в Neon

```bash
pg_restore -h <neon-host> -p 5432 -U <neon-user> -d <neon-db> --no-owner --no-privileges pre-vercel-YYYY-MM-DD.dump
```

Полный дамп восстанавливает схему, данные и sequence. Проверить:

```sql
SELECT count(*) FROM orders;  -- сверка с VPS
SELECT count(*) FROM files;
```

## 3. Vercel

1. vercel.com → Add New Project → импорт репозитория с GitHub (main)
2. Env vars проекта: `DATABASE_URL` (Neon pooled), `AUTH_LOGIN`, `AUTH_PASSWORD`, `JWT_SECRET`, `CLOUDINARY_CLOUD_NAME`, `CLOUDINARY_API_KEY`, `CLOUDINARY_API_SECRET`
3. Deploy → сайт на `<project>.vercel.app`

## 4. Проверка из РФ

С домашнего интернета и с мобильного полный цикл: логин → создать заказ → загрузить файл → CSV-экспорт.

## 5. Домен

1. Vercel → Domains → добавить домен
2. В DNS (Cloudflare, DNS-only): A-запись `@` → `76.76.21.21`
3. SSL выпустится автоматически

## Откат на VPS

1. DNS обратно на IP VPS — сайт снова на VPS через минуты
2. Заказы, созданные на Neon в окно тестирования, вернуть на VPS полным дампом:

```bash
pg_dump -h <neon-host> -U <neon-user> -d <neon-db> -Fc -f neon-latest.dump
pg_restore -h 127.0.0.1 -U <vps-user> -d <vps-db> --clean --if-exists --no-owner --no-privileges neon-latest.dump
```

## 6. Финальная уборка (через ~месяц стабильной работы)

- Отключить GitHub Actions (`deploy.yml`) — чтобы пуши не дёргали VPS
- Отключить VPS у хостера
- Бэкапы Neon: у free-тарифа нет PITR, раз в месяц `pg_dump` себе на комп

## Возможные грабли

- **Prisma «can't find query engine»** на Vercel — лечится `includeFiles` в `vercel.json` (уже добавлено); если не помогло, зафиксировать runtime: `"functions": { "api/index.js": { "runtime": "nodejs20.x" } }`
- **Ошибка региона `fra1`** на тарифе Hobby — убрать `"regions"` из `vercel.json` (функции поедут на iad1, латентность до БД чуть выше)
- **Холодный старт**: Neon засыпает через 5 мин простоя — первый запрос после паузы ~1–2 сек, дальше мгновенно
