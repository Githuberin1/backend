# Build Progress — NEXUS Content Platform

Phase-by-phase tracker. **Do not delete working code from earlier phases.**

Stack: Next.js + TS (frontend) · FastAPI + Python (backend) · Supabase (Auth +
Postgres) · Redis · object storage · Cloudflare · FFmpeg. See SETUP.md for env.

---

## Phase 1 — Foundation ✅ (auth re-architected to Supabase Auth)

### Design system + scaffold + themed shell + home feed (infinite scroll)
Unchanged. Backend `/api/content` = placeholder seed feed (real content in P2).

### Auth — now Supabase Auth (custom FastAPI auth removed)
- **Supabase** handles signup, login, email verification, password reset, sessions.
- Backend verifies the Supabase JWT (`core/supabase_auth.py`, HS256 legacy secret)
  and loads the app profile/role.
- DB: `0002_supabase_auth.sql` — `profiles` keyed to `auth.users` + trigger
  auto-creating a profile on signup + `roles` + RLS (public read; writes via API).
- Backend: `models.py` (Role, Profile), `deps.py` (get_current_user/require_admin),
  `services/user_service.py`, `schemas/user.py`,
  routes `users.py` (`/me`, PATCH `/me`, `/me/become-creator`) +
  `admin.py` (`/admin/users`, PATCH `/admin/users/{id}/role`).
- Frontend: `lib/supabase.ts`, rewritten `auth-provider.tsx` (Supabase session →
  backend profile), login/register/forgot/reset pages via Supabase, header with
  role + "Become a creator". Removed custom verify-email page/banner + old SMTP.
- Removed: custom `security.py`, `email.py`, `email_templates.py`, `auth_service.py`,
  `auth.py`, sessions/users tables.

### Roles
user (default) · creator (self-serve `become-creator`, `ALLOW_SELF_CREATOR`) ·
admin (seed first via SQL, then admin sets others). Admin never self-selectable.

---

## Phase 2 — Content & Taxonomy (NEXT)
Real content model, categories, tags, upload (creator-gated), replaces seed feed.

## Phases 3–12 — see spec / memory.

---

## Phase 2 · Step 0 — Navigation & responsive ✅
- Full sidebar nav (Discover/Trending/Live/Categories/Vault/Favorites/Loved/
  Timeline/Upload*/Shop/VIP/Gems). Mobile: hamburger opens it as a drawer.
- YouTube-style ProfileMenu (dropdown on desktop, bottom sheet on mobile) with
  ALL account options: Profile, Creator Dashboard*, Upload*, Admin**, Become-a-
  creator, Gems, VIP, Shop, Messages, Notifications, Settings, Help, Feedback,
  Log out. Opens from header avatar AND mobile bottom-nav profile tab.
- MobileBottomNav links now work (Discover/Reels/Upload/Messages/Profile).
- Route stubs (ComingSoon) for every destination so nothing 404s.
- React Query refetch-on-focus/reconnect for auto-refreshing data.
  (* creator/admin only, ** admin only)

---

## Phase 2 · Step 1 — Content model + taxonomy ✅
- DB `0003_content.sql`: categories (3-level self-ref), tags, content, content_tags
  + feed index + RLS (public read of published/public).
- Backend models: Category, Tag, Content, ContentTag.
- Services `content_service.py`: slugify, DB feed (cursor on published_at,id),
  create content (+ get-or-create tags), formatting (views/time/duration/tier).
- Routes: `/api/content` now serves **real DB feed** (replaced seed) + POST create
  (creator/admin) + `/content/mine`; `/api/categories` (+admin create);
  `/api/tags` (+admin create).
- Frontend: real **Upload** page (creator-gated form: type/category/tags/media URL/
  thumbnail/visibility/premium+gems), **Categories** page (from DB), feed empty state.
- Media = external URL for now (storage abstraction ready for Supabase Storage/B2 next).

### Next in Phase 2
- Admin category/tag management UI. Supabase Storage upload for files/thumbnails.
- Content detail/watch page. Real category filtering on the home pills.

---

## Phase 2 · Step 2 — Watch page + admin taxonomy + real pills ✅
- Backend: `GET /api/content/{id}` detail (media/creator/category/tags) + view
  increment; ContentDetailOut schema.
- Frontend: content **watch/detail page** `/content/[id]` (YouTube embed / <video> /
  image / link, premium locked state, tags, creator). Cards now link to it.
- **Admin page** (`/admin`, admin-only): create + list categories and tags.
- Home **category pills now load real categories** from DB and filter the feed.
- Fixed dark styling of native <select> dropdowns (color-scheme: dark).

### Next
- Supabase Storage upload (real files/thumbnails). Content edit/delete + admin
  moderation. Category reorder/hierarchy UI.

---

## Phase 2 · Step 2b — YouTube-style watch page ✅
- `/content/[id]` rebuilt to match YouTube: player (full-bleed on mobile),
  title, channel row + Follow, like/dislike/share(copies link)/save/download
  actions, expandable description box with tags/category, and an "Up next"
  related list (right sidebar on desktop, stacked below on mobile).
- Fixed Next-14 params crash and 16:9 thumbnails on all devices (earlier).
- Like/Follow/Save are visual for now — wired to real backend in Phase 5 (social).

---

## Phase 2 · Step 2c + Phase 5(part1) ✅
- Watch page: **sidebar auto-collapses** on /content/* (like YouTube) so the
  player is always large & consistent on every video/device.
- DB `0004_social.sql`: likes, follows. Models Like/Follow. Optional-auth dep.
- Backend: POST /content/{id}/like (toggle), POST /users/{id}/follow (toggle);
  content detail now returns like_count/is_liked/follower_count/is_following/creator_id.
- Watch page: **Like and Follow are now real** (optimistic + persisted, login
  prompt if signed out, own-content hides Follow). Save/Download still visual.

---

## Phase 5(part2) — Comments ✅ (watch page now fully YouTube-like)
- DB `0005_comments.sql`: comments (2-level via parent_id) + comment_likes.
- Backend: list (top/newest, pinned-first), create, replies, like toggle,
  delete (author/content-owner/admin), pin (content-owner/admin).
  Content detail returns comment_count.
- Frontend `components/content/comments.tsx`: comment box, list, like, reply
  (nested), view/hide replies, pin, delete, Top/Newest sort, login prompt.
  Mounted on the watch page under the description.
- Anonymous comments (no-signup) come later behind a feature flag.

---

## Watch page — full YouTube parity + fixes ✅
- **Fixed intermittent crash**: related list had the same React-Query key as the
  home infinite feed (shape clash) → gave it its own key ["related-feed", id].
- **Dislike** works (0006_dislikes.sql) — mutually exclusive with like.
- **Realtime-ish**: comments poll every 5s, related every 15s, refetch on focus —
  new data appears automatically (live-chat feel).
- **Description**: URLs + #hashtags highlighted (links clickable), tag chips cyan.
- **Creator → channel**: avatar/name link to /channel/[id]. New functional
  channel page (public profile: avatar/cover/name/verified/followers/posts +
  Follow + their content grid). Backend: GET /users/{id}, /users/{id}/content.
- **Counts YouTube-style** everywhere (1K/1.2M/3B) via Intl compact + server B.

---

## Upload page — real file upload + storage abstraction + edit/delete ✅
- Backend `core/storage.py`: StorageProvider interface. SupabaseStorageProvider
  (signed upload URLs). B2StorageProvider stub — switch with STORAGE_PROVIDER=b2,
  no frontend change. `POST /uploads/sign` returns a provider-agnostic signed
  PUT target + public URL. httpx added.
- Backend: PATCH /content/{id} + DELETE /content/{id} (owner/admin);
  detail now returns visibility.
- Frontend `lib/storage.ts`: uploadFile (XHR progress) + readVideoDuration.
- Upload page: **drag & drop** or select → real upload to Supabase Storage with
  progress, auto title/duration, optional thumbnail upload, link fallback, then
  metadata + publish.
- Content **edit** page (/content/[id]/edit) + **Edit/Delete** on the watch page
  for owner/admin.

### Setup note
Create a **public Storage bucket named `media`** in Supabase, and set backend env
`SUPABASE_SERVICE_ROLE_KEY` (Settings → API). STORAGE_PROVIDER=supabase (default).

### Next
- Custom YouTube-style video player (branded controls) for direct/HLS video.
- B2 + Cloudflare + FFmpeg/HLS for no-buffer streaming (later).

---

## Creator channel/profile — full YouTube-style ✅
- DB `0007_profile_channel.sql`: content.is_short, profiles.links (jsonb).
- Backend: public profile returns cover, links, website, location, total_views,
  joined, follower/content counts; content list has ?kind=video|short|photo;
  PATCH /users/me accepts cover_url + links; upload sets is_short.
- Upload auto-detects **Shorts** (vertical video ≤3min) → Shorts tab.
- Channel page (/channel/[id]): banner + avatar + name + verified + @username ·
  followers · posts + bio "…more" **About modal** (Description / Links / More info:
  joined, followers, posts, total views, Share, Report). Tabs **Videos / Shorts /
  Photos / About**. Owner sees **Customize channel**; others see **Follow**.
  Only the owner/admin can edit — everyone else is read-only.
- Channel editor (/channel/edit): upload banner + avatar, edit name/username/bio/
  location/website + add/remove links. Banner uses object-cover (adjusts like YT).
- Profile menu "Your channel" → /channel/[me]. Clicking a creator anywhere opens
  their channel.

---

## Realtime messaging + feature flags ✅
- DB `0008_messaging.sql`: feature_flags (seed messages=on), conversations (1:1),
  messages.
- Backend: feature_service + require_feature("messages") gate. message_service
  (get-or-create conv, list, messages, send, read). Routes:
  POST/GET /conversations, GET/POST /conversations/{id}/messages,
  POST /conversations/{id}/read. GET /features (public). Admin:
  GET /admin/features, PATCH /admin/features/{key}.
- Frontend: /messages Messenger-style (list + chat), realtime polling
  (messages 2.5s, list 5s), read receipts. **Message** button on every channel
  next to Follow (login prompt if signed out). useFeatures hook gates the
  Message button + Messages nav (mobile bar, profile menu). Admin page has a
  **Feature controls** toggle — turning "messages" off hides it site-wide and
  blocks the APIs.

---

## Realtime everywhere + fixes ✅
- Supabase **Realtime** enabled (0009 messages/conversations, 0010 comments/
  likes/dislikes/comment_likes) with read RLS so the browser client gets instant
  row pushes. Messages + comments now update instantly (polling kept as fallback).
- Messages page rebuilt Messenger-style (full-height, aligned bubbles, avatars,
  pinned input, responsive; bottom nav hidden on /messages).
- **Google Drive** (and Vimeo) links now embed & play on the watch page.
- Logged-out: tapping the profile icon shows a **sign-in panel** (Log in / Sign up).

### Next
- **Anonymous like/comment/reply** (no signup) + admin toggle — needs schema
  changes (nullable user + anon_id on likes/comments/comment_likes) and a
  server-issued anonymous identity; doing it carefully next so existing
  engagement doesn't break.

---

## Anonymous like/comment/reply ✅
- DB `0011_anonymous.sql`: likes/dislikes/comment_likes get surrogate id PK +
  nullable user_id + anon_id (existing rows untouched); comments get anon_id +
  anon_name. Feature flags anonymous_likes / anonymous_comments (default on).
- Backend: signed anonymous-id cookie (core/anon.py, ANON_SECRET) + get_actor
  dependency (user OR anon). likes/dislikes/comments/comment-likes are all
  actor-aware; routes gate anon actions by the flags (401 → prompt login).
- Frontend: api() now sends credentials (anon cookie) + exposes error.status.
  Watch like/dislike work as guest (401 → login). Comments: guests can comment/
  reply with an optional name; 401 → login. Admin toggles hide/allow accordingly.

---

## Custom video player ✅ (direct/uploaded video)
- `components/content/video-player.tsx`: play/pause, scrub bar (buffered+
  progress+thumb), time, volume slider+mute, playback speed menu, fullscreen,
  auto-hide controls, big center play, double-tap ±10s seek.
- Keyboard: space/k play, j/l ±10s, ←/→ ±5s, ↑/↓ volume, m mute, f fullscreen,
  0-9 seek %.
- Mobile gestures: swipe up → next video, swipe down → immersive/fullscreen
  (video only), double-tap sides ±10s.
- **Lock** button: freezes all controls (playing continues) until unlocked.
- `onNext` auto-advances to first related on end / skip / swipe-up.
- Applies to direct video files; YouTube/Drive/Vimeo keep their own embed player.
  (Skipped the theater/embed option per request.)

---

## Player v2 + fixes ✅
- get_detail wrapped in try/except for anon like/dislike (won't 500 if 0011 not
  applied) — fixes "content not found" for guests.
- Player: full keyboard (space/k, j/l, arrows, m, f, home/end, 0-9, </> speed,
  ,/. frame step when paused), settings menu (Sleep timer / Playback speed /
  Quality-Auto) like YouTube, mobile gestures (double-tap accumulate ±10, single
  tap toggles controls, horizontal drag = scrub with time bubble, swipe up = next,
  swipe down = fullscreen), lock. Skipped theater/miniplayer/captions/chapters
  (no data / per request).

---

## Player v3 ✅
- Loop, Miniplayer (Picture-in-Picture), Copy video URL added to the settings
  menu + right-click context menu (works on mobile via the gear).
- Keyboard `i` = miniplayer/PiP. Loop toggles from menu.
- **Keyboard shortcuts now global** on the watch page (no need to click the
  video first); ignored while typing in inputs/textarea/contenteditable.

---

## Player v4 + Save + no-download ✅
- Save works: DB `0012_saves.sql` + toggle_save/list_saved, POST /content/{id}/save,
  GET /content/saved, detail is_saved. Watch page Save button toggles; **My Vault**
  (/library) lists saved content. **Download button removed.**
- No download: video `controlsList=nodownload` + right-click disabled on the
  player (context menu does nothing) — the obvious download paths are gone.
- Player: settings menu is a **bottom sheet on mobile** (no longer off-screen);
  fullscreen requests **landscape** on mobile; swipe up = fullscreen (or, in
  fullscreen, reveal related panel from the bottom via ^chevron / swipe up),
  swipe down = hide panel / exit fullscreen.
- **End screen**: when a video ends it shows related videos + Replay (no forced
  auto-advance). Loop skips the end screen.
- (Note: true download-prevention isn't 100% possible on the web; obvious paths
  removed. OG link-preview + embeddable player is the next step.)

---

## Player touch fix + link preview + embeddable player ✅
- Player wrap now `touch-none` → mobile swipes control the player (no page
  scroll / pull-to-refresh). Controls restyled (bigger tap targets, hover pills).
- Watch page split: server `page.tsx` with **generateMetadata** (OG + Twitter
  player card: title, description, og:image=thumbnail, og:video → /embed) and
  client `components/content/watch-client.tsx`. Pasting a NEXUS link anywhere now
  unfurls a preview card.
- **Embeddable player** `/embed/[id]` (no shell, full-bleed) — other sites can
  iframe it to play the video (YouTube-style embed). Direct video → custom
  player; YouTube/Drive/Vimeo → their iframe.
- New env: NEXT_PUBLIC_SITE_URL (this site's public URL, for preview/embed links).

---

## oEmbed + Timeline ✅
- **oEmbed** provider `GET /api/oembed?url=<site>/content/<id>` returns the
  iframe embed HTML; content page head exposes the oEmbed discovery <link>. So
  platforms that support oEmbed (WordPress, Discord, etc.) auto-embed the player
  from a pasted link — no manual iframe. (Requires NEXT_PUBLIC_SITE_URL set.)
- **Timeline** (`/history`): DB `0013_history.sql` (watch_history upserted when a
  logged-in user opens a video), `GET /content/history`, page lists recently
  watched. My Vault (/library) = saved.
- Note for embedding on other sites: use the **/embed/<id>** URL (or let oEmbed/OG
  handle it) — pasting the plain /content/ URL into a raw iframe shows the full
  page; the embed URL shows only the player.

---

## Embed-only-player + Favorites/Loved ✅
- Watch page detects it's inside an iframe (window.self !== window.top) and
  redirects to /embed/[id] → embedding the /content/ URL now shows ONLY the
  player, not the full site.
- Loved (/liked) = liked content (GET /content/liked). Favorites (/favorites) =
  saved/bookmarked content (GET /content/saved). Both real now.

---

## Guest merge + remove My Vault ✅
- Removed My Vault from the sidebar (Favorites already covers saved).
- Guest→account merge: POST /users/merge-anon reassigns the guest's anon likes/
  dislikes/comment-likes/comments to the logged-in account (drops rows that would
  duplicate existing reactions), then clears the anon cookie. Frontend shows a
  one-time "Bring your guest activity? Merge / Start fresh" prompt after login
  when an anon cookie is present.

---

## Performance + Notifications ✅
- **DB pooling**: replaced NullPool with a reusing async pool (pool_size 5,
  overflow 10, pre-ping, recycle) — no more fresh SSL connection per request.
  Kept SQLAlchemy compiled-cache ON; only asyncpg server-prepared cache disabled
  (pgbouncer-safe via unique statement names). Big latency win.
- `/api/ping` (GET+HEAD, no DB) for UptimeRobot keep-alive; root also accepts HEAD.
- Lowered polling (realtime covers instant): comments 20s, messages 15s, convo
  list 20s, related 30s.
- **Notifications** (`0014_notifications.sql`): created on follow, comment, reply,
  comment-like (recipient = content owner / parent author / comment author; no
  self-notify). Routes: GET /notifications, /unread-count, POST /read-all,
  /{id}/read. Page `/notifications` (read/unread, mark-all-read, click→open).
  Header bell shows a live unread badge.
- Biggest real-world speedups are infra: put Render + Supabase in the SAME region,
  and keep the service always-on (paid or UptimeRobot).

---

## Playlist system ✅ (Phase 6)
- DB `0015_playlists.sql`: playlists (title/description/visibility public|unlisted|
  private) + playlist_items (many-to-many, ordered by position). RLS.
- Backend: create/rename/edit/delete, add/remove item, reorder, list-mine
  (with ?content_id → contains flag), public list, detail (visibility-respecting).
  Routes under /playlists + GET /users/{id}/playlists.
- Frontend: **Playlists** page (grid + create), **playlist detail** (Play all,
  reorder ↑↓, remove, change visibility, delete — owner only), **Add-to-playlist**
  modal on the watch page ("Playlist" button; checkbox each playlist; create new).
  Sidebar gained a Playlists link.

---

## Channel playlists tab + design polish ✅
- Channel page: new **Playlists** tab showing the creator's public playlists.
- Added reusable **PageHeader** + **EmptyState** (icon tile + title + subtitle +
  optional action). Applied to Playlists / Loved / Favorites / Timeline pages and
  refreshed the ComingSoon stub — cleaner, more polished look.

---

## Performance pass (YouTube-style) ✅
- get_detail: ~10 sequential DB round-trips → **ONE combined query** (counts,
  flags, tags via subqueries). Huge win on a remote DB.
- **View/history split out**: get_detail no longer writes; a separate
  POST /content/{id}/view beacon (fired once by the watch page) increments views
  + upserts history. So the metadata read is cheap and safe to prefetch/cache.
- **In-memory TTL cache** for hot public reads: feed (15s), categories (60s,
  invalidated on create), features (30s, invalidated on admin toggle).
- Frontend: React Query staleTime 60s + gcTime 5m + refetchOnWindowFocus off
  (far fewer refetches); **prefetch video detail on card hover** → clicking opens
  instantly (YouTube-style). Next.js already prefetches links in viewport.
- Biggest remaining win is infra: colocate Render + Supabase in one region and
  keep the API warm (UptimeRobot /api/ping) — do that in the dashboards.

---

## Fixes: counts, message notif, upload, messages UI, player ✅
- **Counts fixed**: reverted get_detail to reliable individual count queries
  (the one-query optimization was silently failing → like/comment/follower showed
  0). No rollback on error (that was expiring the content object).
- **New-message notification**: sending a message notifies the recipient (bell).
- **Upload = video only** now (accept video/*), nicer PageHeader.
- **Messages page** full-width layout (no cramped centered box), cleaner panes.
- **Player settings menu** z-index fixed (its backdrop was covering the menu on
  desktop, so items didn't click); fullscreen kept. Embeds use the provider's
  own controls (YouTube/Drive iframe).
- Channel banner/header spacing + min-height on tab content (consistent sizing).

---

## Search system ✅ (Phase, spec 32)
- DB `0016_search.sql`: search_history (recent/trending/zero-result) + pg_trgm
  index on content.title.
- Backend: smart keyword search (stopword-aware EN+BN, weighted scoring across
  title/tags/category/creator/description), filters (type, category, free/premium),
  sorting (relevance/newest/oldest/most_viewed/most_liked/most_discussed). Every
  search is logged (captures zero-result queries for admin). Endpoints:
  GET /search, /search/suggestions, /search/trending, /search/recent,
  DELETE /search/recent/{id}. Suggestions + trending cached.
- Frontend: header **SearchBar** — 300ms debounce, live suggestions, recent
  (deletable) + trending when empty; Enter/click → /search. **/search** results
  page with type/category/free/sort filters (URL-driven) + result grid.
- NOTE: also includes the earlier upload isVideo build fix.
- Next: Messages block/report + basic moderation; Redis as an optional cache/rate
  layer (falls back to in-memory when REDIS_URL unset).

---

## Search fix + instant-feel ✅
- Search was returning nothing because the array-param query failed silently.
  Rewrote with reliable per-keyword ILIKE params (title/tags/category/creator/
  description weighted); route now never 500s (returns empty on error).
- N+1 check: owner/category/role are already lazy="joined" → feed & cards load
  in one query (no N+1). Confirmed.
- Watch page now renders a full-layout **skeleton** immediately (video + title +
  meta + related), so opening feels instant while data streams in. Combined with
  hover-prefetch, a hovered thumbnail opens with ~no wait.

---

## YouTube-style cards + instant open ✅
- ContentCard redesigned to YouTube layout: thumbnail on top, then avatar +
  title (2 lines) + channel (verified tick) + views·time BELOW it (no on-thumb
  overlays). **Single tap opens it** everywhere → the mobile double-tap is gone.
  PC hover lifts the card slightly (image-3 style); no video-preview (kept light).
  Applies to feed, search, channel, playlists, liked/favorites/history.
- Instant open: added route-level **loading.tsx** for /content/[id] — Next shows
  the watch skeleton the instant a thumbnail is clicked, then the page fills in
  (video → like/comment/related). generateMetadata (OG preview) kept, now with a
  5s timeout so a cold API can't stall the loading screen.
- Embed unchanged and still works.

---

## Posts / Feed system ✅ (Facebook-style) + mobile search
- Mobile search fixed: header search icon → /search, which now has its own
  mobile search input.
- DB `0017_posts.sql`: posts (title, body, images jsonb, video_url, repost_of) +
  post_likes / post_saves / post_comments. RLS.
- Backend: create/delete post, feed (cursor infinite), user posts, like, save,
  repost, comment; notifications on like/comment/repost (+follow). Anyone can post.
- Frontend: **/feed** (create box: title, body, multiple image upload, video URL;
  infinite scroll) + **PostCard** (link auto-highlight, video embed/play,
  image grid, Like/Comment/Repost/Share/Save/Follow, embedded repost) +
  **/post/[id]**. **Photos tab → Posts tab** on channels. Mobile bottom-nav center
  is now **Feed** (video upload stays in the profile menu); PC sidebar gained Feed.

---

## Posts polish ✅
- Long body truncates to 3 lines with See more / See less.
- Media priority: if a post has photos, photos show (+ a "Watch video" card when
  a video URL is also present, opens in a modal); if no photos, the video previews
  inline.
- Facebook-style photo lightbox: click a photo → fullscreen viewer with
  click-to-zoom and prev/next.
- Mobile Feed icon is now a flat nav item (like Reels), not the raised center.

---

## Feed v2: mixed ranked feed + post edit + collage ✅
- **Mixed ranked feed** at /feed: posts AND videos blended by a hot score
  (engagement + recency decay) — popular items surface, fresh items still appear,
  so a refresh brings up new posts. GET /feed?offset= (offset infinite scroll),
  staleTime 0 so each visit re-ranks. Videos render as compact cards, posts as
  PostCard.
- **Post edit**: PATCH /posts/{id}; owner gets a pencil → inline edit (title, body,
  video URL, add/remove images).
- **Facebook-style photo collage**: 1 full / 2 side-by-side / 3 (big left + 2
  stacked) / 4+ (2x2, 4th shows +N).
- **Saved/liked posts** now show in Favorites (Saved posts) and Loved (Liked
  posts) alongside videos. GET /posts/saved, /posts/liked.
- Timeline endpoints confirmed correctly ordered (work once logged in + 0013 run).
- No new migration — uses existing 0017 tables.

---

## Feed refinements + post modal ✅
- Feed is **posts only** now (videos removed from /feed; they live in the video
  sections). Ranked by engagement + recency with a per-user shuffle → each user's
  feed differs and a refresh surfaces newer posts.
- Single image in a post is **centered** (object-contain); **#hashtags** highlight
  (link to search); links highlight.
- **Post comments now support replies + likes** (like the video comments):
  `0018_post_comments2.sql` (parent_id + post_comment_likes). Threaded list with
  Like/Reply, reply notifications.
- **Facebook-style post modal**: the comment button opens an overlay showing the
  full post (text + photos) and threaded comments with a composer; **X** to close.
- **Search** now also finds posts by title/body → a Posts section on /search.
- Video cards show "N views • time".

---

## Modal/lightbox portal fix + more images ✅
- Overlays (post modal, photo lightbox, video modal) now render through a React
  **Portal to <body>**, escaping transformed/animated ancestors — so fixed
  positioning, full-page blur, and z-order work correctly (no more "floats inside
  the post" or "photo goes under other posts").
- Post modal restructured: proper centered flex-column (header / scrollable post +
  comments / sticky composer), page fully blurred + click/scroll captured behind it.
- Lightbox X and prev/next are now solid, clear, sized buttons with a counter.
- Image cap raised 4 → 10 (collage shows first 4 + +N; lightbox shows all).

---

## Following + fast feed + zoom + Messenger redesign ✅
- **Following list**: GET /users/{id}/following + a /following page (sidebar link)
  with unfollow. Live Streams removed from the sidebar.
- **Feed loads fast** now: posts_feed batched (rank query + one load + batched
  viewer flags) — no per-post N+1, quick like the video feed.
- **Photo zoom from any point**: lightbox click zooms toward the click position
  (transform-origin set from the pointer), 2.2x.
- **Messenger redesigned** to the Messenger layout (dark theme, responsive):
  chat list with search + big avatars + online dots + unread (bold + blue dot) +
  last message · time; chat header with active/inactive status; message bubbles
  (👍 shows large); **emoji picker**; send button becomes a 👍 like when empty.
  Online/active uses Supabase Realtime **presence**. Real backend + realtime kept.

---

## Messages full-width fix + VIP/Gems monetization ✅
- Messages: chat area now fills via **absolute positioning** inside a relative
  main — guaranteed full width (no empty right), responsive kept.
- **Monetization** (`0019_monetization.sql`): wallets + gem_ledger (atomic credit),
  configurable vip_tiers (silver/gold/platinum/lifetime, seeded), vip_subscriptions
  (auto-expire via expires_at, checked on read), gem_packages, orders (manual
  payment), payment_settings (admin), daily_claims (idempotent per day).
- Flow: VIP with a gem price is bought instantly from the Gem balance (atomic
  debit + activate). Money-priced (lifetime) + Gem packages use **manual payment**:
  user submits a transaction ID → an admin **Approves** (idempotent, atomic credit/
  activate) or Rejects. Daily bonus for VIPs, once per day.
- Endpoints: /vip/tiers, /vip/me, /vip/subscribe, /wallet, /wallet/daily-bonus,
  /gems/packages, /payment/settings, /orders(+ /me), admin: /admin/orders,
  /admin/orders/{id}/approve|reject, /admin/payment-settings.
- Frontend: **/vip** (tiers + status + subscribe/buy), **/gems** (balance, buy,
  daily bonus, history), shared **PaymentModal**, and admin **orders approval +
  payment settings**. Provider-abstraction ready (manual now; Stripe/Crypto later
  finalize only via server-side approval — success pages never credit directly).

---

## Admin-configurable monetization + real header balance ✅
- Header Gems balance is now **live from the DB** (/api/wallet); VIP pill shows the
  real active tier. Shop removed from the sidebar.
- **Admin fully controls VIP + Gems** from /admin: create/edit/delete VIP tiers
  (name, gems price, USD price, duration, daily gems, benefits, rank, active) and
  gem packages (label, gems, price, active) — all persisted in the DB. Endpoints:
  GET/POST/PUT/DELETE /admin/vip-tiers and /admin/gem-packages.
- Gems rules honored: balance never trusted from client (server-computed), all
  changes atomic (single-statement wallet update + ledger row), auditable ledger.

---

## Monetization fixes ✅
- **Approve now works**: rewrote atomic _credit into two simple statements (the
  repeated bind-param broke on asyncpg); approve wraps its fulfilment in a
  rollback-on-error and surfaces a clear message instead of a silent 500.
- /wallet and /vip/me are resilient (never 500 → return safe defaults).
- Profile menu Gems balance is now **live from /api/wallet** (was hardcoded).
- **Shop page deleted** entirely (removed from nav earlier; page gone now).
- Admin shows **Saved ✓** confirmations (payment settings + tier/package changes),
  and tier/package edits also refresh the public /vip and /gems views.

---

## Fixes: approve cast bug, messenger theme, mobile nav, shop ✅
- **Approve/view bug fixed**: SQLAlchemy text() mis-parsed :param::uuid (double
  colon → phantom :uuid param). Replaced all with cast(:x as uuid). This fixes
  order approval AND view counting / watch-history (record_view used the same).
- Messenger re-themed to the site's palette (deep bg, white text, cyan→violet
  gradient sent bubbles) instead of zinc.
- Mobile bottom nav now shows on the messages page (with bottom padding so the
  composer sits above it). Shop removed from the profile menu too.

---

## Reels + Premium unlock ✅
- **Reels** (/reels): vertical swipe feed (scroll-snap, autoplay the visible one),
  reusing the content architecture (is_short videos). Like / comment / follow /
  share; premium reels show a lock + Unlock button. Mobile-nav-aware, full-bleed.
  Backend GET /reels (cursor).
- **Premium unlock** (`0020_content_unlocks.sql`): server-side flow — verify user,
  premium, Gems balance → **atomic deduct + ledger + unlock record** →
  POST /content/{id}/unlock. get_detail now withholds media_url when locked and
  returns is_locked; watch page shows a working Unlock (X Gems) button; reels too.

---

## Reels polish + Shorts row ✅
- Reels **comment** now opens a Facebook-style **CommentPanel** (portal): a
  right-side panel on desktop, a bottom sheet on mobile (reuses the full video
  comment system with replies/likes) — no more navigation/crash.
- Reels **double-tap to like** with a heart burst; single tap play/pause.
- **PC sidebar** gained a **Reels** entry.
- **Home Shorts row**: a horizontal 9:16 Reels strip on the home page (links to
  /reels). Shorts are **excluded from the main 16:9 feed** (list_feed filters
  is_short) so vertical shorts don't render as wide cards.
- Note: opening a short from a channel/search still uses the watch page for now;
  a reels-deep-link (/reels?v=id) can route those to the vertical player next.

---

## Reels everywhere: interspersed rows, ranked feed, auto-thumb, deep-link ✅
- Home feed now **interleaves Reels rows at random intervals** (first after ~6-10
  videos, then random gaps), each row a different set of shorts. Shorts stay out
  of the 16:9 grid.
- **Reels feed is ranked + personalized** (views/likes/comments + recency + a
  per-user hash jitter) — not identical for everyone; a refresh reshuffles.
  GET /reels now takes a limit.
- **Auto-thumbnail**: if no thumbnail is chosen on upload, a frame is grabbed
  from the video (canvas) and uploaded automatically.
- **Deep-link**: short cards (home/channel/search) link to **/reels?v=ID**, which
  opens that short first in the vertical player. is_short is now on the card model.

---

## Reels/feed polish + notifications + VIP badge ✅
- Home grid capped at 4 videos/row; Reels rows show 6 (grid); removed the Hero
  premiere poster.
- Channel gained a **Reels tab** (9:16 grid → opens /reels?v=ID).
- Reels player: poster + preload + tap-to-play overlay so high-res shorts start
  reliably; never opens in the 16:9 player (deep-linked).
- Premium reels/videos: Unlock button (lifetime, via content_unlocks) — already
  permanent.
- **Order-approved notification** to the buyer (Gems/VIP). **VIP badge** now on
  the channel header (public profile returns the active tier).

---

## VIP badges + notification thumbnails + category art + videos-tab fix ✅
- Channel Videos tab now excludes reels (kind accepts plurals: videos/reels/photos).
- **VIP badge** now on feed cards and comment authors too (batched active_vip_map;
  Creator + CommentAuthor carry vip_tier) — plus channel/header already.
- **Notification details**: shows who (actor) + a thumbnail — content thumbnail
  for comment/reply/like, follower avatar for follows (`0021_notif_image.sql`).
- **Category cards** show the thumbnail of the top-engagement video in that
  category (views·0.1 + likes·2 + comments·3), cached with the category list.

---

## Notifications fix + autoplay + Creator Studio ✅
- Post notifications now carry the **actor's name + a thumbnail** (post image /
  content thumb / follower avatar) — no more "Someone"; proper per-type icons
  (repost/like/comment/order).
- Watch page **autoplays muted** on open so a clicked video starts immediately
  (unmute from controls).
- **Creator Studio** (/creator/dashboard) — real Content management tab: filter by
  All/Videos/Reels/Photos/Premium/Free/Published/Private; per-item performance
  (views/likes/comments); quick controls for visibility, premium+price, comments
  on/off, **download permission**; Edit (tags/category) + Delete.
  Backend: GET /content/manage, extended PATCH (status, allow_comments,
  allow_download, is_short), `0022_allow_download.sql`.

---

## Notifications YouTube-style + purchase/expiry + dashboard Posts ✅
- Notifications now show the **actor's profile pic** (left) + **content thumbnail**
  (right) + the content/post **title** in the message (`0023_notif_actor.sql`).
- New events: **content purchase/unlock** notifies the owner ("X unlocked your
  premium content …"); **VIP expiry** notifies the user once (lazy check on
  /vip/me). Icons for purchase/vip_expired added.
- Creator Studio: **Photos → Posts** — manage your posts (performance + Open +
  Delete) alongside videos/reels.

---

## VIP everywhere + Trending + msg edit/delete + reels/unlock ✅
- **VIP crown everywhere**: post authors, reels creators, message list/header,
  video cards, comments, channel/header (batched active_vip_map on each payload).
- **Trending** page + GET /content/trending: ranks published videos by engagement
  (views·0.2 + likes·4 + comments·6) with recency decay; cached.
- **Message edit/delete** (Messenger-style): PATCH/DELETE message endpoints +
  hover edit/delete on your own messages.
- **Unlock revenue**: gems spent unlocking premium content are now credited to
  the content owner's wallet (content_sale in the ledger).
- Reels: double-tap-to-like fixed (dedicated tap layer so rail buttons still
  work), video autoPlays; deep-linked reel opens first.

---

## Reels deep-link fix + Help/Feedback ✅
- Reels: fixed the deep-link jumping to the first reel — when /reels?v=ID is used
  we wait for that reel and disable scroll-anchoring, so the clicked reel opens
  first. Increased per-user jitter so feeds differ more between users.
- **Help Center** (/help) — written FAQ covering uploads, Reels, Feed/posts,
  Gems & VIP, premium unlock, messaging, playlists, Creator Studio, trending,
  notifications, troubleshooting.
- **Feedback** (/feedback) — submit a message (category + optional email);
  stored in `0024_feedback.sql`. Admin sees all feedback in /admin.

---

## Admin Phase 10 ✅
- **User management**: role change + ban/unban (banned users blocked at auth).
- **Content moderation**: report button on videos + posts (POST /report) → admin
  **Reports queue** with preview → Dismiss or **Take down** (hides content /
  deletes comment/post).
- **Analytics dashboard**: users, creators, content, posts, comments, likes,
  views, gems in circulation, VIP active, pending orders, open reports.
- **Audit log**: every admin action (ban, role, takedown, order approve, site
  update) recorded and shown.
- **Site branding**: admin sets name / logo / favicon / tagline in the DB
  (GET /site, PUT /admin/site); the header now shows the DB site name + logo.
- `0025_admin.sql`.

---

## Admin tabs + Report/Block ✅
- Admin panel is now **tabbed** (Overview / Users / Reports / Content / Orders /
  Monetization / Features / Site / Feedback / Audit) — only the active tab loads
  its data (mount/enabled-gated), much cleaner.
- **Report everywhere**: videos, posts, **comments, users, messages** all report
  into the same admin queue (generic target_type). Admin queue shows previews for
  every type; **Take down** removes content / deletes comment·post·message /
  bans a reported user.
- **Block users** (`0026_blocks.sql`): Block/Unblock on a channel; blocked pairs
  can't message each other; profile returns is_blocked; /users/me/blocked list.

---

## Storage → Backblaze B2 ✅
- Implemented B2StorageProvider (S3-compatible, boto3 presigned PUT). Browser
  uploads straight to B2; backend only signs. Reads via public bucket or
  STORAGE_PUBLIC_BASE (Cloudflare/CDN). Set STORAGE_PROVIDER=b2 + B2_* env.
- Frontend unchanged (upload flow is provider-agnostic). boto3 added to requirements.
- Next: Phase 4 processing — Step A (worker registry + admin panel + job creation).

---

## Admin fixes: crash guard, share, moderation notify + unhide ✅
- Added a route-group error boundary so a component error shows a friendly
  "Try again" instead of a white "Application error" screen.
- Fixed channel/profile share: native share sheet where available, else copy
  link with a "Copied!" confirmation.
- Report takedown of content now **notifies the owner**; admin can **Restore
  (unhide)** from a new **Hidden** tab (owner notified again).
- Report queue rows have a 1-click **View** link to open the reported item.
- Admin panel kept fully tabbed + responsive (wrapping rows, scrollable tabs).

---

## Takedown fix + watch nav crash + admin responsive ✅
- Takedown 500 fixed: moderation notifications now queue on the same transaction
  (add, not notify) so hide/restore commit atomically.
- Content→content navigation no longer flashes the error page: the watch query
  keeps previous data during navigation and auto-retries cold-start API blips.
- Admin panel made fully responsive (reports/users/audit/orders rows stack on
  mobile, actions wrap, tabs scroll) for phone/tablet/desktop.

---

## Takedown hard-fix + Settings + reviews + own-content + Docker ✅
- **Takedown 500 fixed for good**: core action (hide/delete/ban + resolve report)
  commits FIRST with only existing columns; the owner notification and audit-log
  write are separate best-effort commits, so a missing audit_log table can never
  500 the takedown again. Same core-first pattern for ban/restore/resolve/site.
- **Settings page** (/settings): account info, **change password** (Supabase),
  **sign out of all devices**, and **login history** (device · IP · time). A login
  event is recorded on every sign-in.
- **Reviews / star ratings** (`0027`): members leave a 1–5★ review; the **Gems
  page** shows the average, count and recent reviews + a leave-a-review box.
- **Own content edit/delete**: your own channel now shows a **Manage content**
  button (Creator Studio) and inline edit/delete on each of your video cards.
- **Docker / VPS (Phase 12)**: backend + frontend Dockerfiles, docker-compose
  (with redis), and DEPLOY-DOCKER.md.

### Run these SQL migrations (in order) if you haven't:
0025_admin.sql, 0026_blocks.sql, 0027_settings_reviews.sql

---

## YouTube-style mini-player + unmute + stability ✅
- Global **mini-player**: minimize button (and `i`) shrinks the video to a
  corner player that keeps playing across pages; **Maximize** returns to the
  watch page, **X** closes it. Works on desktop and mobile (sits above the
  mobile nav). Replaces the old native-PiP "minimize" that couldn't return or
  close.
- 16:9 watch player no longer force-mutes — video plays with **sound by default**;
  the user mutes if they want.
- Shorts (9:16) open in the **reels** player from every page (loved/saved/etc.)
  via is_short routing + the fixed reels deep-link.
- Global query retry w/ backoff so Render cold-starts recover instead of erroring
  when content loads.

---

## Mini-player resume + reels routing/save + upload reels + admin tabs ✅
- Mini-player now resumes from the exact timestamp both ways: minimize keeps the
  current time, and Maximize returns to the watch page continuing from where the
  mini left off (not from the start).
- Root cause of "9:16 opens in 16:9": upload only marked a video as a Reel when
  vertical AND <=3min AND not a link. Now ANY vertical file is a Reel, links can
  be marked via a new "This is a Reel" toggle, and Creator Studio has a
  Reel/Video toggle to fix existing videos. Shorts then route to the Reels player
  everywhere (home, loved, saved, search) via ContentCard is_short routing.
- Reels: added a **Save/bookmark** button (is_saved surfaced in the reels feed);
  saved reels appear in Favorites and open back in the Reels player.
- Admin tabs now wrap (no cut-off "Audit") instead of horizontal scroll.
