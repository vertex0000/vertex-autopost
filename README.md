# Vertex Autopost — Client Dashboard

Clean, minimal dashboard for managing Instagram autoposting. Clients never see n8n.
Single file (`index.html`), no build step, ready for GitHub Pages.

---

## 1. GitHub Pages pe host karo

1. github.com → **New repository** → naam: `vertex-autopost` → **Public** → Create.
2. **Add file → Upload files** → `index.html` aur `README.md` upload karo → **Commit changes**.
3. Repo → **Settings → Pages** → Source: **Deploy from a branch** → Branch: `main` / `(root)` → **Save**.
4. 1-2 minute baad link milega: `https://<username>.github.io/vertex-autopost/`

Abhi dashboard **demo mode** mein hai: data browser (localStorage) mein save hota hai.

---

## 2. Pages (page structure)

| Page | Kya hai | Brief se mapping |
|---|---|---|
| **Overview** | Next post preview + countdown, At a glance, Next 7 days, Recent activity | Dashboard Overview, Automation Status, Posts Scheduled/Published, Next Scheduled Post |
| **Content** | Tabs: Media library (upload, filter), Captions (template editor + live preview + AI captions toggle), Hashtags (groups, 30-limit) | Media Library, Caption Editor, Hashtag Library |
| **Queue** | Ordered list, drag & drop + arrows, slot time per post | Post Queue Preview |
| **Schedule** | Weekly time slots, time zone, month calendar with thumbnails | Calendar View, Time Slots, Recurring Schedule |
| **Analytics** | KPI strip, reach chart, best time to post, top posts (7/30/90 days) | Analytics Overview, Engagement Stats |
| **Activity log** | Filter (Done / Fixed itself / Needs you), search, CSV download | Error Logs |
| **Settings** | Profile, Instagram connection, posting rules, notifications, reset | User Settings |

### Add-ons jo brief ke upar jode
- **Approve each post first** mode (client har post OK kare)
- **AI captions from image** toggle
- **Post now / Move to end** on the next post
- **Queue running low** warning + "Queue lasts until" date
- **Best time to post** insight
- **Instagram disconnected** state with reconnect
- **CSV export** of activity log
- Plain-language log messages (client ko samajh aaye, technical error nahi)
- Mobile drawer navigation, keyboard focus, reduced-motion support

---

## 3. Component hierarchy

```
App
├── Sidebar
│   ├── Brand
│   ├── Nav (count badges: queue size, errors)
│   └── AccountCard (IG handle, connection, demo badge)
├── Topbar
│   ├── MenuButton (mobile)
│   ├── AutopostingPill (global on/off)
│   └── Avatar
└── View (hash router: #/overview …)
    ├── Overview → NextPostCard (PhonePreview, Countdown, Facts, Actions) · GlanceRows · WeekStrip · ActivityFeed
    ├── Content  → Tabs → MediaLibrary (DropZone, FilterSeg, TileGrid) · CaptionEditor (TemplateList, Editor, VariableChips, Preview) · HashtagGroups
    ├── Queue    → QueueList (QueueRow: grip, position, thumb, caption, slot, controls)
    ├── Schedule → SlotEditor (DayRow × 7, TimeChips, AddTime) · MonthCalendar
    ├── Analytics→ RangeSeg · KpiStrip · ReachChart · BestTimeBars · TopPosts
    ├── Logs     → FilterSeg · Search · LogTable
    └── Settings → Profile · Instagram · PostingRules · Notifications · DemoData
Toast (global, aria-live)
```

## 4. User flow (client)

```
Login → Overview
  → Content: upload images → they join the queue in order
  → Content > Captions: AI captions ON, ya template likho
  → Schedule: time slots set karo (e.g. Mon–Sat 10:00)
  → Queue: order check / drag karke badlo
  → (optional) Settings: "Approve each post first" ON
  → Overview: countdown dekho, "Post now" ya "Approve"
  → Activity log: kuch fail ho toh "Needs you" filter
  → Analytics: kya kaam kar raha hai
```

## 5. Code architecture (index.html ke andar)

| Section | Kaam |
|---|---|
| `CONFIG` | Supabase URL + anon key (Phase 2) |
| `seed()` | Demo data |
| `S` + `save()` | App state + persistence (abhi localStorage) |
| `upcoming(k)` | Schedule slots se agle k post times |
| `vOverview()`, `vContent()` … | Har page ka HTML |
| `render()` | Router + sidebar + view |
| Event delegation (`data-act`) | Saare clicks ek jagah handle |
| `addFiles()` / `shrink()` | Upload + 1080px resize |

---

## 6. Phase 2: Supabase (next step)

Tables (SQL editor mein chalana):

```sql
create table clients (
  id uuid primary key default gen_random_uuid(),
  name text, brand text, ig_handle text, ig_user_id text,
  ig_token text,               -- sirf service role / n8n padhe (RLS)
  automation boolean default true,
  approval boolean default false,
  ai_captions boolean default true,
  caption_language text default 'Hinglish',
  timezone text default 'Asia/Kolkata',
  created_at timestamptz default now()
);

create table media (
  id uuid primary key default gen_random_uuid(),
  client_id uuid references clients(id) on delete cascade,
  position int,                -- queue order
  storage_path text,           -- Supabase Storage (public URL -> Instagram)
  caption text,
  status text default 'queued', -- queued | posted | failed
  approved boolean default true,
  posted_at timestamptz, ig_post_id text
);

create table slots (
  id uuid primary key default gen_random_uuid(),
  client_id uuid references clients(id) on delete cascade,
  weekday text, time text      -- 'Mon', '10:00'
);

create table templates (id uuid primary key default gen_random_uuid(), client_id uuid references clients(id), name text, body text);
create table hashtag_groups (id uuid primary key default gen_random_uuid(), client_id uuid references clients(id), name text, tags text[]);
create table logs (id bigserial primary key, client_id uuid references clients(id), level text, message text, created_at timestamptz default now());
```

**n8n ka naya flow:** har 15 min → Supabase se `clients` (automation = true) → slot time aa gaya? → `media` mein sabse chhota `position` jo `queued` + `approved` ho → caption (AI ya template) → Instagram publish → `media.status = posted` + `logs` mein entry.

Bonus: images Supabase Storage mein hongi, jiska public URL Instagram seedha le leta hai, toh catbox ki zaroorat khatam.
