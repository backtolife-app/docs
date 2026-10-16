# Parental Controls Domain

The parental-control side of the schema has four tables that are easy to confuse. This page explains what each one is for, who writes to it, and how they relate.

## The four tables

```
family_links ─── 1:1 ───▶ parental_controls
     │
     └─── 1:N ──▶ sleep_schedules        (device-wide bedtime lock)

users ─── 1:1 ──▶ strict_schedule_filters (content filters + always_on)
  │
  └─── 1:N ──▶ strict_schedule_blocks  (Focus / All-Day windows)
```

| Table | Scope | Shape | Who writes |
|---|---|---|---|
| `family_links` | Links one parent ↔ one child. Many-to-many overall. | One row per link. | Parent + child (linking flow). |
| `parental_controls` | Per-child parent settings (lock, screen-time caps, allow-list). | One row per link. | Parent app. |
| `sleep_schedules` | **Device-wide** bedtime lock. | Up to **two** rows per link (`bucket = weeknight | weekend`). | Parent app. |
| `strict_schedule_filters` | **Social-media-only** content filters: 30 flags across YouTube, Instagram, X and TikTok, plus the `always_on` list. | One row per child. | Child app (own choices) or parent app (overlay in `parent_strict_schedule_filters`). |
| `strict_schedule_blocks` | **When** the filters lock: Focus windows and All-Day blocks. | One row per `(user_id, kind)`. | Child app or parent app (overlay in `parent_strict_schedule_blocks`). |

The easy mental model: **the strict-schedule tables are about what's visible inside the social apps. `sleep_schedules` is about whether the child's phone works at all.** They do not share data or enforcement code.

---

## Linking a child

1. **Child** hits `POST /child/invite`. Backend generates an 8-character `invite_code`, stores it on a new `family_links` row with `status = pending` and `invite_expires_at = now + 24h`, and returns the code.
2. **Parent** scans the QR (or types the code) → `POST /parent/children/link` with that code.
3. Backend sets `parent_user_id`, flips `status` to `active`, creates the `parental_controls` row, and clears `invite_code`.
4. Parent can now GET the child's dashboard and PUT controls.

Unlinking sets `status = revoked` and cascades nothing — the parental-control and schedule rows are kept for audit. A fresh link creates a new `family_links` row; history is preserved.

---

## Screen-time limits

`parental_controls` has three relevant columns:

- `daily_screen_time_limit_minutes` — **legacy** single value.
- `daily_limit_weekday_minutes` — Mon–Fri cap.
- `daily_limit_weekend_minutes` — Sat–Sun cap.

The parent app always reads and writes the **weekday + weekend** pair. The legacy column is only consulted as a fallback when both new columns are null (e.g. data written by an old client).

Client logic (`ParentalControl.todayLimitMinutes` in `children_model.dart`):

```dart
int? get todayLimitMinutes {
  final isWeekend = DateTime.now().weekday >= DateTime.saturday;
  return (isWeekend ? dailyLimitWeekendMinutes : dailyLimitWeekdayMinutes)
      ?? dailyScreenTimeLimitMinutes;
}
```

The child app is expected to mirror this logic when it enforces the limit. Backend does no enforcement today — it only stores.

---

## Sleep schedules

Unlike `strict_schedule_filters` (a single row per child), `sleep_schedules` is a **multi-row** table with up to two rows per child.

```
sleep_schedules
  bucket=weeknight  enabled=true   22:00 → 07:00  days=[1,2,3,4]
  bucket=weekend    enabled=true   23:00 → 08:30  days=[5,6,7]
```

Why two rows:

- Most families want different bedtimes on school nights vs. weekends.
- The `days` mask is free-form, so a parent who splits differently (e.g. Mon–Sat tight, Sun loose) can still use the two slots however they like.
- More than two buckets would be rare. The `unique(family_link_id, bucket)` constraint enforces the cap.

Upsert by `(family_link_id, bucket)` via `PUT /parent/children/{id}/sleep-schedules`:

```http
PUT /api/parent/children/42/sleep-schedules
Content-Type: application/json

{
  "bucket": "weeknight",
  "enabled": true,
  "start_time": "22:00",
  "end_time": "07:00",
  "days": [1, 2, 3, 4]
}
```

### Cross-midnight windows

`end_time < start_time` means the window crosses midnight. The backend stores both as `TIME` and the client app resolves the date at enforcement time — no special handling on the DB side.

### Days

Days are ISO-8601 weekday integers: `1 = Monday`, `2 = Tuesday`, … `7 = Sunday`. An empty array effectively disables the row (no day matches). Prefer setting `enabled = false` instead.

---

## Always-allowed apps

Stored as a JSON column on `parental_controls`:

```json
"always_allowed_apps": [
  { "name": "Phone",    "emoji": "📞", "package": "com.android.phone" },
  { "name": "Messages", "emoji": "💬" },
  { "name": "Maps",     "emoji": "🗺️" }
]
```

- `name` is required, max 60 chars.
- `emoji` optional, max 8 chars (UTF-8 emoji cluster).
- `package` optional — the Android package id or iOS bundle id once the child app can resolve installed apps. Kept nullable until that enforcement layer is built.

Apps in this list remain unlocked during sleep windows. The list also factors into per-app blocking (Phase B in the plan), but that enforcement is not yet wired.

Written via `PUT /parent/children/{id}/controls` with an `always_allowed_apps` key. Omitting the key leaves the list unchanged; sending `null` clears it.

---

## Strict schedules — the other "schedule" tables

The content filters and the Focus / All-Day windows live in two tables that replaced the legacy single-row `strict_shedules` (dropped on 2026-09-04):

- **`strict_schedule_filters`**: one row per child, upserted on `user_id`. One boolean per content filter, 30 flags across YouTube, Instagram, X and TikTok, listed on the [schema page](schema.md), plus the `always_on` JSON list of the flags applied 24/7.
- **`strict_schedule_blocks`**: one row per `(user_id, kind)`. The Focus windows (`weekday`, `weekend`) with `start_time`, `end_time`, `days`, which define **when the filters lock**, not a device-wide lock; the All-Day blocks (`allday_school`, `allday_weekend`) carry their own `filters` list.
- A parent's choices sit in `parent_strict_schedule_filters` and `parent_strict_schedule_blocks`, merged over the child's own rows on read.

When sleep windows and strict schedules are both configured for a child:

1. **Outside** any sleep window → the content filters apply inside a Focus window, all day under an All-Day block, and always for the `always_on` list; other apps unrestricted.
2. **Inside** a sleep window → everything locked except `always_allowed_apps`, whatever the filters say.

The parent app surfaces them as separate rows on the Rules screen:

- **Sleep schedule** → writes to `sleep_schedules`.
- **Focus schedule, All-Day Schedule and Always-on filters** → write to the strict-schedule tables through `/parent/children/{id}/schedule`.

---

## What's **not** in the schema yet

- **Push command queue** — no table stores pending "lock now" / "unlock" commands. FCM send-side service needs to land first.
- **Per-app quotas** — Individual App Quotas (from the plan) will add a new table (one row per child+app+bucket). Not built yet.
- **Subscription entitlement** — no `subscriptions` table. RevenueCat integration will add either a column on `users` or a dedicated table.
- **Location / geofencing** — no GPS tables. Optional add-on in the plan.

See [PLAN.md](../../../parent/PLAN.md) for the roadmap.
