# NEXUS — Setup Guide (Supabase Auth + FastAPI)

**Architecture now:** Supabase Auth handles signup / login / email verification /
password reset. FastAPI verifies the Supabase token and manages everything else
(profiles, roles, content…). Roles live in our `profiles` table.

---

## 1) Supabase project
Create a project. Note the **Database Password**.

## 2) Run the SQL
Supabase → **SQL Editor** → run these in order:
1. **`backend/db/migrations/0002_supabase_auth.sql`** — roles + profiles + signup trigger.
2. **`backend/db/migrations/0003_content.sql`** — categories, tags, content tables.
3. **`backend/db/migrations/0004_social.sql`** — likes, follows.
4. **`backend/db/migrations/0005_comments.sql`** — comments + comment likes.
5. **`backend/db/migrations/0006_dislikes.sql`** — dislikes.
6. **`backend/db/migrations/0007_profile_channel.sql`** — shorts flag + profile links.
7. **`backend/db/migrations/0008_messaging.sql`** — conversations, messages, feature flags.
8. **`backend/db/migrations/0009_realtime.sql`** — Realtime + RLS for messages.
9. **`backend/db/migrations/0010_realtime_engagement.sql`** — Realtime for comments/likes.
10. **`backend/db/migrations/0011_anonymous.sql`** — anonymous likes/comments.
11. **`backend/db/migrations/0012_saves.sql`** — saved content (My Vault).
12. **`backend/db/migrations/0013_history.sql`** — watch history (Timeline).
13. **`backend/db/migrations/0014_notifications.sql`** — notifications.
14. **`backend/db/migrations/0015_playlists.sql`** — playlists + items.
15. **`backend/db/migrations/0016_search.sql`** — search history + trigram index.
16. **`backend/db/migrations/0017_posts.sql`** — posts / feed (likes, saves, comments, reposts).
17. **`backend/db/migrations/0018_post_comments2.sql`** — post comment replies + likes.
18. **`backend/db/migrations/0019_monetization.sql`** — Gems wallet, VIP tiers, orders, daily bonus.
19. **`backend/db/migrations/0020_content_unlocks.sql`** — premium content unlocks.
20. **`backend/db/migrations/0021_notif_image.sql`** — notification thumbnails.
21. **`backend/db/migrations/0022_allow_download.sql`** — per-content download permission.
22. **`backend/db/migrations/0023_notif_actor.sql`** — notification actor avatar + VIP expiry flag.
23. **`backend/db/migrations/0024_feedback.sql`** — feedback / support messages.
24. **`backend/db/migrations/0025_admin.sql`** — ban, reports, audit log, site settings.
25. **`backend/db/migrations/0026_blocks.sql`** — user blocking.
26. **`backend/db/migrations/0027_settings_reviews.sql`** — login history + reviews.
(Run each new numbered migration as it ships.)

## 3) Enable Email auth
Supabase → **Authentication → Providers → Email** → enabled.
- Dev shortcut: turn **"Confirm email" OFF** to log in without clicking a link.
- Production: keep it ON + configure SMTP (next step).

## 4) Email (now handled by Supabase)
Supabase → **Authentication → Emails → SMTP Settings** → enable custom SMTP:

| Field | Value (Resend) |
|---|---|
| Host | `smtp.resend.com` |
| Port | `465` |
| Username | `resend` |
| Password | your Resend API key (`re_...`) |
| Sender email | `onboarding@resend.dev` (or verified domain) |
| Sender name | `NEXUS` |

> This is now the ONLY place email is configured. Any SMTP provider works.

## 5) URL configuration
Supabase → **Authentication → URL Configuration**:
- **Site URL**: `https://nexus-pro26.vercel.app`
- **Redirect URLs**: add `https://nexus-pro26.vercel.app/**`

## 6) Get keys — Supabase → Settings → API
- **Project URL** → `https://YOUR-REF.supabase.co`
- **anon public** key → frontend
- **JWT Secret** (Legacy) → backend (verifies tokens)

---

## 7) Environment variables

### Backend → Render
| Key | Value |
|---|---|
| `ENVIRONMENT` | `production` |
| `DATABASE_URL` | Supabase connection string (plain `postgresql://…`) |
| `SUPABASE_URL` | `https://YOUR-REF.supabase.co` |
| `SUPABASE_JWT_SECRET` | JWT Secret from Settings → API |
| `CORS_ORIGINS` | `https://nexus-pro26.vercel.app` |
| `ALLOW_SELF_CREATOR` | `true` |
| `PYTHON_VERSION` | `3.12.7` |

### Frontend → Vercel
| Key | Value |
|---|---|
| `NEXT_PUBLIC_API_URL` | Render backend URL (no trailing `/`) |
| `NEXT_PUBLIC_SUPABASE_URL` | `https://YOUR-REF.supabase.co` |
| `NEXT_PUBLIC_SUPABASE_ANON_KEY` | anon public key |

Redeploy both after setting env (Vercel: uncheck build cache).

---

## 8) Roles — user / creator / admin

- **user** — everyone starts here on signup.
- **creator** — a logged-in user clicks **"Become a creator"** (profile menu).
  Instant by default (`ALLOW_SELF_CREATOR=true`).
- **admin** — nobody self-selects (security). Seed the **first admin by SQL**,
  then admins promote others.

**Seed your first admin** (after signing up once):
```sql
update profiles set role_id = (select id from roles where name = 'admin')
where id = (select id from auth.users where email = 'YOUR_EMAIL_HERE');
```

Admin changes roles: `PATCH /api/admin/users/{id}/role` body `{"role":"creator"}`.

---

## 9) Verify
1. `/register` → row in Supabase **auth.users** + auto row in **profiles**.
2. Confirm email (if ON) → redirected back logged in.
3. Header shows avatar + role; "Become a creator" upgrades role.
4. `GET /api/users/me` returns profile + role.


---

## 10) Storage (file uploads)
1. Supabase → **Storage** → create a **public** bucket named **`media`**.
2. Backend env (Render): `SUPABASE_SERVICE_ROLE_KEY` (Settings → API → service_role),
   `STORAGE_PROVIDER=supabase`, `STORAGE_BUCKET=media`.
3. Later (B2/Cloudflare): implement `B2StorageProvider` in `app/core/storage.py`,
   set `STORAGE_PROVIDER=b2` + `STORAGE_PUBLIC_BASE=https://your-cdn`. Frontend unchanged.


---

## 11) Anonymous engagement
- After `0011_anonymous.sql`, guests can like/comment/reply **without signing in**.
- Backend env (Render): set **`ANON_SECRET`** to a long random string
  (signs the anonymous-visitor cookie). Cookie uses the same COOKIE_SECURE /
  COOKIE_SAMESITE as auth (so keep them true/none in prod).
- Admin → Feature controls → toggle **anonymous_likes** / **anonymous_comments**
  off to require login for guests (logged-in users always allowed).


---

## 12) Keep-alive (UptimeRobot)
Point UptimeRobot (or any cron) at **`https://your-backend.onrender.com/api/ping`**
with **HEAD** or GET. It responds instantly (no DB) and keeps the free Render
service from sleeping. (`/api/health` also works and additionally checks config.)

---

## Backblaze B2 storage (replaces Supabase Storage)

The app now uploads straight to **Backblaze B2** via its S3-compatible API
(browser → B2 presigned PUT; nothing large passes through the backend).

**1. Create a B2 bucket** (Public) and an **Application Key** (with read+write to it).

**2. Find your S3 endpoint** on the bucket page, e.g. `s3.us-west-004.backblazeb2.com`.

**3. Set these env vars on Render:**
```
STORAGE_PROVIDER=b2
B2_KEY_ID=<applicationKeyId>
B2_APP_KEY=<applicationKey>
B2_BUCKET=<bucketName>
B2_ENDPOINT=https://s3.us-west-004.backblazeb2.com
B2_REGION=us-west-004
# optional, if you put Cloudflare / a custom domain in front of B2:
# STORAGE_PUBLIC_BASE=https://cdn.yoursite.com
```

**4. Make the bucket public** (so uploaded videos/images are viewable), or put
Cloudflare in front and set `STORAGE_PUBLIC_BASE`.

**5. Set B2 bucket CORS** so the browser can PUT directly. Using the B2 CLI:
```
b2 update-bucket --corsRules '[{"corsRuleName":"web","allowedOrigins":["https://YOUR-FRONTEND.vercel.app"],"allowedOperations":["s3_put","s3_get","s3_head"],"allowedHeaders":["*"],"exposeHeaders":["etag"],"maxAgeSeconds":3600}]' <bucketName> allPublic
```
(add `http://localhost:3000` to allowedOrigins for local dev.)

That's it — `pip install` picks up `boto3`, redeploy, and uploads go to B2.
Existing Supabase-hosted files keep working (their URLs are unchanged).
