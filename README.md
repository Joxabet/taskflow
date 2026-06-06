# TaskFlow

A personal productivity app built with vanilla HTML/CSS/JS and Supabase. Deployed on Vercel at [tasking.fyi](https://www.tasking.fyi).

---

## What it does

- **Tasks** — create tasks with title, notes, tags, subtasks, category, urgency, due date, recurring schedule, owner and group
- **Calendar** — monthly view showing tasks and recurring previews as ghost pills
- **Notes** — multi-note scratchpad with autosave, title and word count
- **Pomodoro** — focus timer with 25min / 5min break / custom modes and sound alert
- **Groups** — collapse tasks into named groups to hide them from the main list
- **Dark / light mode** — persisted per user
- **Import / Export** — full JSON backup and restore (merge or replace)
- **Mobile** — responsive layout with FAB, collapsible header menu, touch-friendly tap targets
- **iOS PWA** — add to home screen from Safari for full-screen app experience

---

## Tech stack

| Layer | Technology |
|---|---|
| Frontend | Vanilla HTML, CSS, JavaScript (single file) |
| Database | Supabase (PostgreSQL) |
| Auth | Supabase Auth (email/password) |
| Hosting | Vercel |
| Domain | tasking.fyi |

---

## Project structure

```
taskflow/
  index.html        ← entire frontend (HTML + CSS + JS)
  icon_1024.png     ← app icon (iOS home screen + favicon)
  vercel.json       ← security headers for Vercel
  README.md         ← this file
```

---

## Supabase setup

### 1. Create a project
Go to [supabase.com](https://supabase.com), create a new project, choose a region and password.

### 2. Run the schema SQL
In **Supabase → SQL Editor**, run the following to create all tables, enable RLS, add policies, grants, and seed defaults for new users:

```sql
-- Enable UUID generation
create extension if not exists "pgcrypto";

-- Tasks
create table if not exists tasks (
  id          text primary key,
  user_id     uuid references auth.users(id) on delete cascade not null,
  title       text not null,
  note        text default '',
  tags        jsonb default '[]',
  end_date    text default 'No date',
  category    text default 'Work',
  urgency     text default 'Medium',
  recurring   text default 'None',
  status      text default 'Todo',
  completed   boolean default false,
  created_at  bigint not null,
  subtasks    jsonb default '[]',
  owner       text default '',
  group_name  text default ''
);

-- Categories
create table if not exists categories (
  id      uuid primary key default gen_random_uuid(),
  user_id uuid references auth.users(id) on delete cascade not null,
  name    text not null,
  unique (user_id, name)
);

-- Settings
create table if not exists settings (
  id      uuid primary key default gen_random_uuid(),
  user_id uuid references auth.users(id) on delete cascade not null,
  key     text not null,
  value   text,
  unique (user_id, key)
);

-- Team members
create table if not exists team_members (
  id       uuid primary key default gen_random_uuid(),
  user_id  uuid references auth.users(id) on delete cascade not null,
  name     text not null,
  initials text,
  color    text,
  unique (user_id, name)
);

-- Notes
create table if not exists notes (
  id          text primary key,
  user_id     uuid references auth.users(id) on delete cascade not null,
  title       text default '',
  content     text default '',
  created_at  timestamptz default now() not null,
  updated_at  timestamptz default now() not null
);

-- RLS
alter table tasks        enable row level security;
alter table categories   enable row level security;
alter table settings     enable row level security;
alter table team_members enable row level security;
alter table notes        enable row level security;

-- Policies
create policy "tasks: own rows"        on tasks        for all using (auth.uid() = user_id);
create policy "categories: own rows"   on categories   for all using (auth.uid() = user_id);
create policy "settings: own rows"     on settings     for all using (auth.uid() = user_id);
create policy "team_members: own rows" on team_members for all using (auth.uid() = user_id);
create policy "notes: own rows"        on notes        for all using (auth.uid() = user_id);

-- Grants
grant select, insert, update, delete on public.tasks        to anon, authenticated, service_role;
grant select, insert, update, delete on public.categories   to anon, authenticated, service_role;
grant select, insert, update, delete on public.settings     to anon, authenticated, service_role;
grant select, insert, update, delete on public.team_members to anon, authenticated, service_role;
grant select, insert, update, delete on public.notes        to anon, authenticated, service_role;

-- Seed defaults when a new user signs up
create or replace function seed_defaults_for_new_user()
returns trigger language plpgsql security definer
set search_path = public as $$
begin
  insert into public.categories (user_id, name) values
    (new.id, 'Work'), (new.id, 'Personal'), (new.id, 'Shopping'),
    (new.id, 'Health'), (new.id, 'Finance'), (new.id, 'Other');
  insert into public.settings (user_id, key, value) values
    (new.id, 'theme', 'light'),
    (new.id, 'notif_enabled', 'true'),
    (new.id, 'notif_day_before', 'true'),
    (new.id, 'notif_day_of', 'true'),
    (new.id, 'notif_overdue', 'false'),
    (new.id, 'notif_time', '09:00'),
    (new.id, 'notif_urgency', 'All');
  return new;
end;
$$;

create or replace trigger on_auth_user_created
  after insert on auth.users
  for each row execute procedure seed_defaults_for_new_user();
```

### 3. Get your API keys
Go to **Supabase → Project Settings → API** and copy:
- **Project URL** — `https://xxxxxxxxxxxx.supabase.co`
- **anon / public key** — starts with `eyJ...`

### 4. Update index.html
Replace the two constants near the top of the `<script>` section:
```js
const SUPABASE_URL = 'your-project-url';
const SUPABASE_ANON_KEY = 'your-anon-key';
```

### 5. Disable new signups
The app is single-user. After creating your account go to:
**Supabase → Authentication → Settings → disable "Enable new user signups"**

---

## Deploying to Vercel

1. Push the repo to GitHub
2. Go to [vercel.com](https://vercel.com) → **Add New Project** → select the repo
3. Click **Deploy** (no build settings needed — it's a static HTML file)
4. Go to **Settings → Domains** → add your custom domain
5. Update DNS records at your registrar as shown by Vercel

---

## iOS — Add to home screen

1. Open **https://www.tasking.fyi** in **Safari** (not Chrome)
2. Tap the **Share** button
3. Tap **"Add to Home Screen"**
4. Tap **Add**

The app opens full screen with no browser bar.

---

## Supabase — future-proofing (grants)

From **October 30, 2026**, Supabase requires explicit grants on all tables. All tables in this project already have grants applied. When adding a new table in the future, always include:

```sql
grant select, insert, update, delete on public.your_table to anon, authenticated, service_role;
alter table public.your_table enable row level security;
```

---

## Changelog

| Version | Date | What changed |
|---|---|---|
| 1.0 | 2025-05 | Initial Electron desktop app |
| 2.0 | 2025-05 | Migrated to web app — Supabase + Vercel |
| 2.1 | 2025-05 | Auth, import/export, dark mode fixes |
| 2.2 | 2025-05 | Bug fixes — recurring tasks, calendar, timezone |
| 2.3 | 2025-05 | Mobile layout, FAB, collapsible header menu |
| 2.4 | 2025-05 | Notes tab (multi-note with autosave) |
| 2.5 | 2025-05 | Pomodoro timer tab |
| 2.6 | 2025-05 | Task groups (collapse/expand) |
| 2.7 | 2025-05 | iOS PWA icon, sort by In Progress |
