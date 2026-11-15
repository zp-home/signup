# GitHub 账号登记页

开会时员工扫码打开，提交「GitHub 用户名 + 真实姓名」，数据写进 Supabase（托管 Postgres）的 `signups` 表。

- `index.html`：收集表（员工手机上打开的页面）
- `qr.html`：投屏用的二维码页，自动指向同目录的 `index.html`

## 一、建表（Supabase，一次性）

1. 去 https://supabase.com 注册并 New project（区域选 Singapore / Tokyo，国内访问快些）
2. 左侧 **SQL Editor**，粘贴执行：

```sql
create table public.signups (
  id              bigint generated always as identity primary key,
  github_username text        not null,
  real_name       text        not null,
  created_at      timestamptz not null default now()
);

-- 同一个用户名（不分大小写）只能登记一次，重复提交页面会提示「已经登记过」
create unique index signups_github_username_key on public.signups (lower(github_username));

-- 前端用的是公开 key，靠 RLS 保证：匿名用户只能插入，不能读、改、删
alter table public.signups enable row level security;

create policy "anon insert only" on public.signups
  for insert to anon
  with check (
    char_length(github_username) between 1 and 39
    and char_length(real_name) between 1 and 50
  );

grant insert on public.signups to anon;
```

3. 左侧 **Project Settings → API**，复制两样东西填进 `index.html` 顶部：
   - `Project URL` → `SUPABASE_URL`
   - `anon public` key（或新版的 `sb_publishable_...` key）→ `SUPABASE_KEY`

   **不要填 `service_role` / `sb_secret_` key**，那个有全部权限，放在公开页面上谁都能拿去删库。

## 二、部署到 GitHub Pages

1. 在 GitHub 新建一个**公开**仓库（免费账号的 Pages 只支持公开仓库），如 `signup`
2. 把 `index.html`、`qr.html` 推到 `main` 分支根目录
3. 仓库 **Settings → Pages → Build and deployment**：Source 选 `Deploy from a branch`，Branch 选 `main` / `/ (root)`，Save
4. 一两分钟后访问：
   - 收集表：`https://<你的用户名>.github.io/signup/`
   - 投屏二维码：`https://<你的用户名>.github.io/signup/qr.html`

## 三、开会时

1. 电脑打开 `qr.html` 投屏，员工扫码
2. **开会前自己先用手机扫一次、提交一条测试数据**，确认能写进表，再到表里删掉

## 四、查看 / 导出

- Supabase 左侧 **Table Editor → signups**：直接看，右上角 Export → CSV
- 或者 SQL Editor：

```sql
select github_username, real_name, created_at
from public.signups
order by created_at;
```

- 有人填错要改：在 Table Editor 里直接改或删那一行（页面端没有修改权限）

## 已知限制

- 提交前会调 GitHub 公开 API 核对用户名是否存在，并显示头像让员工自己确认。该接口未登录时按 IP 限 60 次/小时，会场 Wi-Fi 共用出口 IP 时可能被限流；被限流时只跳过核对、不拦提交，只有明确返回 404 才拦。
- 真实姓名没法校验，乱填也能写进去，会后对一遍名单。
- 公开 key 在页面源码里，理论上有人可以绕过页面直接往表里插垃圾数据（只能插，读不到也删不掉）。开完会如果不再收，到 SQL Editor 执行 `drop policy "anon insert only" on public.signups;` 把入口关掉。
