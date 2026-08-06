# Независимая копия инструмента (своя база данных)

Инструмент — самодостаточный файл `index.html`, но данные (загрузки, контрагенты, журнал действий) хранятся не в нём, а в отдельном облачном проекте Supabase. Чтобы у партнёра была полностью независимая копия (свои данные, не связанные с текущей базой), нужно завести новый Supabase-проект и подключить к нему скопированный файл.

Разово выполняет человек с доступом к email для регистрации — регистрацию аккаунта нельзя сделать автоматически.

## 1. Завести новый проект Supabase

1. Откройте [supabase.com](https://supabase.com) → **New project**.
2. Задайте пароль базы данных (сохраните его в менеджере паролей — он не понадобится в самом инструменте, но нужен для администрирования).
3. Дождитесь создания проекта (~2 минуты).

## 2. Создать таблицы

**SQL Editor → New query**, вставить целиком и выполнить:

```sql
create extension if not exists pgcrypto;

create table public.sb_uploads (
  id uuid primary key default gen_random_uuid(),
  report_date date not null,
  uploaded_by uuid references auth.users(id),
  uploaded_by_email text,
  filename text,
  row_count integer,
  created_at timestamptz not null default now()
);

create table public.sb_snapshots (
  id uuid primary key default gen_random_uuid(),
  upload_id uuid references public.sb_uploads(id) on delete cascade,
  report_date date not null,
  client_raw text not null,
  client_key text not null,
  manager text,
  debt numeric,
  advance numeric,
  deposit numeric,
  equipment_value numeric,
  daily_payment numeric,
  forecast_debt numeric,
  weight_kg numeric,
  overdue_days numeric,
  bucket text,
  created_at timestamptz not null default now()
);
create index sb_snapshots_client_key_idx on public.sb_snapshots(client_key);
create index sb_snapshots_upload_id_idx on public.sb_snapshots(upload_id);
create index sb_snapshots_report_date_idx on public.sb_snapshots(report_date);

create table public.sb_actions (
  id uuid primary key default gen_random_uuid(),
  client_key text not null,
  client_raw text,
  action_type text not null,
  action_date date not null,
  comment text,
  invoice_period_end date,
  payment_amount numeric,
  recovered_amount numeric,
  tons_returned numeric,
  created_by uuid references auth.users(id),
  created_by_email text,
  created_at timestamptz not null default now()
);
create index sb_actions_client_key_idx on public.sb_actions(client_key);
create index sb_actions_action_date_idx on public.sb_actions(action_date);

-- Инструмент только читает и добавляет строки (ничего не изменяет и не удаляет) —
-- поэтому политик update/delete нет специально: это неизменяемый журнал.
alter table public.sb_uploads enable row level security;
alter table public.sb_snapshots enable row level security;
alter table public.sb_actions enable row level security;

create policy "read for authenticated" on public.sb_uploads for select using (auth.role() = 'authenticated');
create policy "insert for authenticated" on public.sb_uploads for insert with check (auth.role() = 'authenticated');

create policy "read for authenticated" on public.sb_snapshots for select using (auth.role() = 'authenticated');
create policy "insert for authenticated" on public.sb_snapshots for insert with check (auth.role() = 'authenticated');

create policy "read for authenticated" on public.sb_actions for select using (auth.role() = 'authenticated');
create policy "insert for authenticated" on public.sb_actions for insert with check (auth.role() = 'authenticated');
```

## 3. Отключить подтверждение email

Инструмент использует логины вида `имя@opalubka.invalid` — на такой адрес не может прийти письмо. **Authentication → Providers → Email** → выключить «Confirm email». Пользователей создавайте вручную: **Authentication → Users → Add user**, либо оставьте кнопку «Зарегистрироваться» в самом инструменте (она создаёт пользователя сразу без подтверждения при выключенной настройке).

## 4. Подключить скопированный файл к новому проекту

1. **Project Settings → API** → скопировать **Project URL** и **anon public** ключ.
2. Открыть скопированный `index.html`, найти в начале `<script>` (сразу после подключения `supabase-js`) строки:
   ```js
   const SUPABASE_URL = "https://khzxfhxpcrjshsssghok.supabase.co";
   const SUPABASE_ANON_KEY = "sb_publishable_rrwlI1VXeWmQJ1Zn3YT3Ow_pUK04-ha";
   ```
3. Заменить оба значения на свои из шага 1. Сохранить файл.

Готово — файл теперь работает с полностью отдельной, независимой базой данных: свои контрагенты, свои загрузки, свой журнал действий, никак не связанные с исходным инструментом.

## Перенос уже накопленных данных (необязательно)

Если нужно перенести текущие данные (а не начать с чистой базы) — в исходном Supabase-проекте: **Table Editor** → каждая таблица → **Export as CSV**, затем в новом проекте **Table Editor** → **Import data from CSV** для тех же трёх таблиц (сначала `sb_uploads`, затем `sb_snapshots` и `sb_actions`, чтобы внешние ключи `upload_id` совпали).
