# ADAL TARGET — Жиналыс кестесі (Supabase нұсқасы)

Мидл таргетологтар мен РОМ-ға арналған күндік/айлық отчет дашборды. `adal_dashboard_supabase.html` — толық жұмыс істейтін нұсқа: логин/тіркелу экраны, рөл жүйесі (мидл/РОМ), барлық дерек Supabase-те сақталады.

## 1. Supabase жобасын жасау

1. [supabase.com](https://supabase.com) → жаңа жоба (тегін жоспар жеткілікті)
2. **Authentication → Providers → Email** → **"Confirm email"** өшіріңіз (ішкі команда құралы үшін email растау артық қадам)
3. **SQL Editor** ашып, төмендегі SQL-дың бәрін бір рет іске қосыңыз

```sql
-- ── Кестелер ─────────────────────────────────────────────────────────────
create table users (
  id uuid primary key references auth.users(id) on delete cascade,
  email text not null,
  name text,
  role text not null default 'midl' check (role in ('midl','rom')),
  created_at timestamptz default now()
);

create table projects (
  id uuid primary key default gen_random_uuid(),
  name text not null,
  created_at timestamptz default now()
);

create table entries (
  date date not null,
  project_id uuid not null references projects(id) on delete cascade,
  revenue numeric default 0,
  cost numeric default 0,
  problem text default '',
  updated_at timestamptz default now(),
  primary key (date, project_id)
);

create table monthly_plans (
  month_key text not null,
  project_id uuid not null references projects(id) on delete cascade,
  plan numeric default 0,
  primary key (month_key, project_id)
);

-- ── "Мен РОМ-мын ба?" тексеру функциясы (RLS ішінде рекурсияны болдырмау үшін) ──
create or replace function is_rom() returns boolean as $$
  select exists (select 1 from users where id = auth.uid() and role = 'rom');
$$ language sql security definer;

-- ── Row Level Security ───────────────────────────────────────────────────
alter table users enable row level security;
alter table projects enable row level security;
alter table entries enable row level security;
alter table monthly_plans enable row level security;

-- users: барлығы оқи алады (рөлдер тізімі үшін), әркім тек өзінің жолын
-- жасай алады (алғашқы кіруде), тек РОМ рөлдерді өзгерте алады
create policy "users_select_all" on users for select using (auth.role() = 'authenticated');
create policy "users_insert_self" on users for insert with check (auth.uid() = id);
create policy "users_update_rom_only" on users for update using (is_rom());

-- projects: барлығы оқиды, тек РОМ жазады/өшіреді
create policy "projects_select_all" on projects for select using (auth.role() = 'authenticated');
create policy "projects_write_rom_only" on projects for insert with check (is_rom());
create policy "projects_update_rom_only" on projects for update using (is_rom());
create policy "projects_delete_rom_only" on projects for delete using (is_rom());

-- entries: барлығы оқиды/жазады (мидл өз күнделігін толтырады)
create policy "entries_select_all" on entries for select using (auth.role() = 'authenticated');
create policy "entries_write_all" on entries for insert with check (auth.role() = 'authenticated');
create policy "entries_update_all" on entries for update using (auth.role() = 'authenticated');

-- monthly_plans: барлығы оқиды, тек РОМ жазады
create policy "plans_select_all" on monthly_plans for select using (auth.role() = 'authenticated');
create policy "plans_write_rom_only" on monthly_plans for insert with check (is_rom());
create policy "plans_update_rom_only" on monthly_plans for update using (is_rom());
```

4. **Settings → API** бөлімінен **Project URL** мен **anon public key**-ды көшіріп алыңыз

## 2. Файлды баптау

`adal_dashboard_supabase.html` файлын ашып, жоғарғы жағындағы екі жолды өз мәндеріңізбен ауыстырыңыз:

```js
const SUPABASE_URL = 'https://YOUR_PROJECT.supabase.co';
const SUPABASE_ANON_KEY = 'YOUR_ANON_KEY';
```

## 3. Бірінші РОМ-ды тағайындау

1. Файлды ашып, өзіңіз **тіркеліңіз** (email + құпия сөз)
2. Supabase-те **Table Editor → users** кестесіне өтіп, өз жолыңызды табыңыз
3. `role` бағанын `midl`-дан **`rom`**-ға қолмен өзгертіп сақтаңыз
4. Сайтты қайта ашсаңыз — сіз енді РОМ, «Проекттер» беті көрінеді

Бұдан кейінгі барлық жаңа тіркелген адам әдепкі бойынша **мидл** болады. РОМ оларды «Проекттер → Пайдаланушылар мен рөлдер» бетінен «РОМ ету» батырмасымен жоғарылата алады.

## 4. GitHub + Vercel-ге орналастыру

`github_guide.html` файлындағы қадамдарды қараңыз (браузермен немесе git арқылы) — процесс бірдей, тек енді файл толық жұмыс істейді, себебі Supabase байланысы дайын.

## Deploy алдындағы тексеру чек-парағы

- [ ] Supabase жобасы жасалды
- [ ] "Confirm email" өшірілді
- [ ] SQL толық іске қосылды (4 кесте + функция + policy-лер)
- [ ] `SUPABASE_URL` / `SUPABASE_ANON_KEY` файлда ауыстырылды
- [ ] Бірінші тіркеліп, `users` кестесінде өз рөліңізді `rom` еттіңіз
- [ ] GitHub-қа салынды, Vercel-ге қосылды
