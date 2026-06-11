# Schema Reference

Every table in the backend, grouped by domain, reflecting the migrations as applied (not the historical create-shapes). Column types follow Laravel's schema builder (`$table->id()` produces `bigint unsigned auto_increment`, `$table->foreignId(...)` produces `bigint unsigned` with an FK constraint, etc.).

Unless noted, each table has `created_at` + `updated_at` (`timestamps()`).

!!! note
    The deprecated single-row `strict_shedules` table (typo intentional) is kept in sync from the canonical `strict_schedule_blocks` + `strict_schedule_filters` tables for older clients. New filters are added to the canonical tables only. See [Strict schedule (child)](#strict-schedule-child).

---

## Identity & auth

### `users`

The single user table for every role. Admin, standard User (child), and Parent are differentiated by the `role` column. Email is unique per role (the `unique_email_per_role_in_users_table` migration, 2026-03-24, relaxed the global unique so one address can register once per role, e.g. parent + child).

| Column | Type | Notes |
|---|---|---|
| id | bigint PK | |
| name | string nullable | |
| surname | string nullable | *(Added 2026-06-05)* |
| date_of_birth | date nullable | *(Added 2026-06-05)* |
| email | string nullable (unique per role) | |
| email_verified_at | timestamp nullable | |
| password | string nullable | Bcrypt. |
| avatar | text nullable | URL. |
| terms | boolean default `false` | T&C accepted. |
| google_id | string nullable | *(Added 2026-01-24)* |
| is_admin_created | boolean default `false` | |
| device_id | string nullable | Originally used for device pinning. |
| language | string nullable | `en` \| `es` \| … |
| timezone | string(64) nullable | *(Added 2026-04-29)* IANA name. |
| provider_id | string nullable (indexed) | Social-auth provider (`google`, `apple`). |
| provider_user_id | string nullable (indexed) | External user id from the provider. |
| reset_code | string nullable | OTP for password reset. |
| reset_code_expires_at | timestamp nullable | |
| role | enum (`Admin`, `User`, `Parent`) default `User` | |
| status | enum (`Pending`, `Rejected`, `Active`, `Deactivate`, `Banned`) default `Active` | |
| remember_token | string | |
| timestamps + softDeletes | | |

!!! note
    Battery fields (`battery_level`, `battery_charging`, `battery_updated_at`) were briefly on `users` but moved to [`devices`](#devices) on 2026-05-21. They no longer exist on `users`.

### `personal_access_tokens`

Laravel Sanctum's token store (the `laravel/sanctum` default migration). The mobile apps authenticate with these bearer tokens.

### `password_reset_tokens`, `sessions`

Laravel defaults. `sessions` backs the cookie-session auth used by the admin panel.

---

## Family & parental controls

See [Parental Controls Domain](parental-controls.md) for the conceptual guide.

### `family_links`

One row per child, after the multi-parent refactor (2026-05-18). Parents are attached through the `family_link_parents` join, not a column on this table.

| Column | Type | Notes |
|---|---|---|
| id | bigint PK | |
| child_user_id | FK → `users.id` cascade delete (unique) | One link per child. |
| app_blocking_globally_enabled | boolean default `false` | Master switch for device app blocking. |
| timestamps | | |

!!! note
    The original `parent_user_id`, `status`, `invite_code`, `invite_expires_at` columns were removed by the refactor (invites moved to `family_link_invites`, parents to `family_link_parents`). The `passcode_hash` / `passcode_set_at` columns that briefly lived here moved to [`parent_passcodes`](#parent_passcodes) on 2026-06-02.

### `family_link_parents` *(2026-05-18)*

Join table linking parents to a `family_link` (many parents per child).

| Column | Type | Notes |
|---|---|---|
| family_link_id | FK → `family_links.id` cascade delete | Composite PK part. |
| parent_user_id | FK → `users.id` cascade delete (indexed) | Composite PK part. |
| added_at | timestamp `useCurrent()` | |
| **primary** | `(family_link_id, parent_user_id)` | No surrogate id, no timestamps. |

### `family_link_invites` *(2026-05-18)*

Pending child-to-parent link invitations (8-char-ish code redeemed by a parent).

| Column | Type | Notes |
|---|---|---|
| id | bigint PK | |
| child_user_id | FK → `users.id` cascade delete | The inviting child. |
| code | string(32) (indexed) | |
| expires_at | timestamp (indexed) | Typically 24 h. |
| used_at | timestamp nullable | Null until redeemed. |
| used_by_parent_id | FK → `users.id` nullable | Parent who redeemed it. |
| timestamps | | |

### `parental_controls`

One row per `family_link`. Holds the parent's configured restrictions for that child.

| Column | Type | Notes |
|---|---|---|
| id | bigint PK | |
| family_link_id | FK → `family_links.id` cascade delete | Effectively 1:1. |
| daily_screen_time_limit_minutes | int nullable | **Legacy single-value column**, kept for back-compat. |
| daily_limit_weekday_minutes | int nullable | *(Added 2026-04-16)* Mon–Fri daily cap. |
| daily_limit_weekend_minutes | int nullable | *(Added 2026-04-16)* Sat–Sun daily cap. |
| always_allowed_apps | json nullable | *(Added 2026-04-16)* Array of `{ name, emoji?, package? }`. Stay unlocked during sleep windows. |
| notification_preferences | json nullable | *(Added 2026-04-17)* Per-child notification toggles + quiet hours. |
| timestamps | | |

!!! note
    `settings_locked` was dropped on 2026-05-28; the lock state is no longer stored here.

### `parent_passcodes` *(2026-06-02)*

The parent's own app passcode (one per parent). Replaces the `passcode_hash` columns that briefly lived on `family_links`.

| Column | Type | Notes |
|---|---|---|
| id | bigint PK | |
| parent_user_id | FK → `users.id` | |
| passcode_hash | string(255) | Hashed PIN. |
| passcode_set_at | timestamp | |
| timestamps | | |

### `sleep_schedules` *(2026-04-16)*

Device-wide "bedtime lock" windows. Up to two rows per child: one for weeknights, one for weekend nights.

| Column | Type | Notes |
|---|---|---|
| id | bigint PK | |
| family_link_id | FK → `family_links.id` cascade delete | |
| bucket | string (`weeknight` \| `weekend`) | |
| enabled | boolean default `true` | |
| start_time | time | `HH:MM:SS` in the stored timezone. |
| end_time | time | If `< start_time` the window crosses midnight. |
| days | json | Array of ISO weekday ints (1 = Mon … 7 = Sun). |
| timezone | string nullable | IANA name. |
| timestamps | | |
| **unique** | `(family_link_id, bucket)` | Max 2 rows per child. |

---

## Devices & app blocking

### `devices` *(2026-05-18)*

One row per registered device per user. Carries the battery telemetry (moved here from `users` on 2026-05-21).

| Column | Type | Notes |
|---|---|---|
| id | bigint PK | |
| user_id | FK → `users.id` cascade delete | |
| device_identifier | string | Stable per-install id. |
| platform | enum (`ios`, `android`) | |
| os_version | string nullable | |
| app_version | string nullable | |
| last_seen_at | timestamp nullable (indexed) | Drives the retention "last seen" signal. |
| battery_level | unsigned tinyint nullable | 0–100. |
| battery_charging | boolean nullable | |
| battery_updated_at | timestamp nullable | |
| timestamps | | |
| **unique** | `(user_id, device_identifier)` | |

### `device_app_blocking` *(2026-05-18)*

Per-device blocking state + setup status.

| Column | Type | Notes |
|---|---|---|
| id | bigint PK | |
| device_id | FK → `devices.id` cascade delete (unique) | One row per device. |
| state | enum | Blocking lifecycle state. |
| platform | enum (`ios`, `android`) | |
| last_synced_at | timestamp nullable | |
| setup_completed_at | timestamp nullable | |
| timestamps | | |

### `device_managed_apps` *(2026-05-18)*

The apps under management on a device. The `*_shielded` columns were renamed to `*_blocked` on 2026-05-21.

| Column | Type | Notes |
|---|---|---|
| id | bigint PK | |
| device_id | FK → `devices.id` cascade delete | |
| identifier | string | iOS label token or Android package name. |
| type | enum (`ios_label`, `android_package`) | |
| app_label | string | |
| icon_url | string nullable | |
| parent_blocked | boolean default `false` (indexed) | Set by the parent. |
| parent_blocked_updated_by | FK → `users.id` nullable | |
| parent_blocked_updated_at | timestamp nullable | |
| self_blocked | boolean default `false` (indexed) | Set by the child (self-control). |
| self_blocked_updated_at | timestamp nullable | |
| last_seen_at | timestamp nullable | |
| last_verified_at | timestamp nullable (indexed) | *(Added 2026-05-19)* |
| timestamps | | |
| **unique** | `(device_id, identifier)` | |

### `device_unblock_requests` *(2026-05-18)*

A child's request to temporarily unblock a managed app, awaiting parent approval.

| Column | Type | Notes |
|---|---|---|
| id | bigint PK | |
| device_id | FK → `devices.id` cascade delete | |
| managed_app_id | FK → `device_managed_apps.id` cascade delete | |
| child_user_id | FK → `users.id` cascade delete | |
| duration_seconds | unsigned int | Requested unblock duration. |
| state | enum (`pending`, `approved`, `denied`, `expired`) | |
| requested_at | timestamp `useCurrent()` | |
| resolved_at | timestamp nullable | |
| resolved_by_parent_id | FK → `users.id` nullable | |
| timestamps | | |

### `device_managed_app_events` *(2026-05-19)*

Append-only audit log of block/unblock actions on managed apps.

| Column | Type | Notes |
|---|---|---|
| id | bigint PK | |
| device_id | FK → `devices.id` | |
| identifier | string | |
| app_label | string | |
| action | enum | Block / unblock / etc. |
| reason | string(255) nullable | |
| actor_user_id | FK → `users.id` nullable | Parent or child who acted. |
| created_at | timestamp `useCurrent()` | No `updated_at`. |

---

## Strict schedule (child)

The canonical strict-schedule data lives in two tables (`strict_schedule_blocks` + `strict_schedule_filters`). The legacy single-row `strict_shedules` table is a deprecated mirror kept in sync for older clients.

### `strict_schedule_blocks` *(2026-04-28)*

Per-kind schedule windows (weekday + weekend), one row each.

| Column | Type | Notes |
|---|---|---|
| id | bigint PK | |
| user_id | FK → `users.id` cascade delete | |
| kind | string(16) | `weekday` \| `weekend`. |
| is_active | boolean default `false` | |
| start_time | time nullable | |
| end_time | time nullable | May cross midnight. |
| days | json nullable | |
| timezone | string(64) nullable | |
| last_start_notified_date | date nullable | De-dupes push notifications. |
| last_end_notified_date | date nullable | |
| timestamps | | |
| **unique** | `(user_id, kind)` | |
| **index** | `(user_id, is_active)` | |

### `strict_schedule_filters` *(2026-04-28)*

Per-user content-filter flags. One row per user. All boolean columns `NOT NULL DEFAULT false`. This is the canonical home for every filter; new filters are added here only.

| Column group | Columns |
|---|---|
| YouTube | `youtube_shorts`, `youtube_comments`, `youtube_subscriptions` *(2026-04-30)*, `youtube_home` *(2026-04-30)* |
| Instagram | `instagram_reels`, `instagram_explore`, `instagram_hide_explore` *(2026-05-11)*, `instagram_hide_suggestions` *(2026-05-11)*, `instagram_likeComments`, `instagram_story`, `instagramDmOnly` |
| Twitter / X *(2026-05-11)* | `twitter_for_you`, `twitter_following`, `twitter_grok`, `twitter_notifications`, `twitter_messages` |
| TikTok *(2026-06-09)* | `tiktok_for_you`, `tiktok_dm_only`, `tiktok_search_only`, `tiktok_notifications` |

Plus `id`, `user_id` (FK, unique), `timezone` (string(64) nullable), `timestamps`.

### `strict_streak_days` *(2026-04-29)*

One row per child per day, recording whether the schedule was scheduled and honored. Backs the streak counter.

| Column | Type | Notes |
|---|---|---|
| id | bigint PK | |
| user_id | FK → `users.id` cascade delete | |
| date | date | |
| was_scheduled | boolean | Whether a block was active that day. |
| was_honored | boolean | Whether the child kept it on. |
| timestamps | | |
| **unique** | `(user_id, date)` | |
| **index** | `(user_id, date, was_scheduled)` | |

### `strict_shedules` *(legacy mirror)*

Deprecated single-row-per-child filter + window. **Named `strict_shedules` (missing c) intentionally — do not "fix" the typo without a rename migration.** Kept in sync from the canonical tables for older clients. Twitter / TikTok filters are deliberately NOT mirrored here (no legacy client exposes them).

| Column | Type | Notes |
|---|---|---|
| id | bigint PK | |
| user_id | FK → `users.id` cascade delete nullable | |
| device_id | string nullable | |
| timezone | string nullable | |
| youtube_shorts, youtube_comments | boolean nullable | |
| instagram_reels, instagram_explore, instagram_likeComments, instagram_story, instagramDmOnly | boolean nullable | |
| start_time, end_time | time nullable | |
| days | json nullable | |
| is_active | boolean nullable | |
| last_start_notified_date, last_end_notified_date | date nullable | |
| timestamps | | |

!!! note
    `instagram_autoplay_off` was removed from this table on 2026-05-14; `instagram_hide_explore` / `instagram_hide_suggestions` were also added then dropped. Autoplay-off is now a per-app device preference, not a schedule filter.

### `strict_data`

Session-level audit trail of when each (legacy) strict schedule became active / was disabled.

| Column | Type |
|---|---|
| id | bigint PK |
| strict_id | FK → `strict_shedules.id` cascade delete |
| started_at | datetime nullable |
| disabled_at | datetime nullable |
| timestamps | |

### `app_statuses`

Per-child snapshot of the social-media filter toggles independent of the schedule window. Mirrors the seven Instagram/YouTube booleans on `strict_shedules`.

| Column | Type |
|---|---|
| id | bigint PK |
| user_id | FK → `users.id` cascade delete nullable |
| device_id | string nullable |
| youtube_shorts, youtube_comments | boolean nullable |
| instagram_reels, instagram_explore, instagram_likeComments, instagram_story, instagramDmOnly | boolean nullable |
| timestamps | |

---

## Strict schedule (parent-managed)

Parent-set schedules for a child, mirroring the child tables. Written by the parent app via `PUT /parent/children/{id}/schedule`.

### `parent_strict_schedule_blocks` *(2026-05-28)*

Identical shape to [`strict_schedule_blocks`](#strict_schedule_blocks-2026-04-28).

### `parent_strict_schedule_filters` *(2026-05-28)*

Identical shape to [`strict_schedule_filters`](#strict_schedule_filters-2026-04-28), including the YouTube/Instagram/Twitter columns at creation and the `tiktok_*` columns added 2026-06-09.

---

## Activity & usage

### `activities`

Session-level login/logout log per user. `login_time` is the "user was active" signal used by the Retention dashboard.

| Column | Type |
|---|---|
| id | bigint PK |
| user_id | FK → `users.id` cascade delete |
| device_id | string nullable |
| login_time | timestamp nullable |
| logout_time | timestamp nullable |
| total_time | string nullable (HH:MM:SS) |
| timestamps | |

### `screen_time_daily` *(2026-03-31)*

Per-app, per-day totals reported by the child app. This is what the parent dashboard bar chart and the in-app analytics read. Replaces the older `screen_usages` table.

| Column | Type | Notes |
|---|---|---|
| id | bigint PK | |
| user_id | FK → `users.id` cascade delete | |
| date | date | Local date in the child's timezone (client-supplied). |
| timezone | string(64) | IANA name. |
| app | string(32) | One of `youtube` \| `instagram` \| `twitter` \| `tiktok`. |
| source | string(16) | `webview` (tracked inside BackToLife) \| `native` (UsageStats / DeviceActivity). |
| seconds | unsigned int default 0 | Day's total. |
| updated_at | timestamp | `useCurrent()->useCurrentOnUpdate()`. No `created_at`. |
| **unique** | `(user_id, date, app, source)` | Upsert key. |
| **index** | `(user_id, date)` | |

### `screen_usages` *(legacy)*

Pre-`screen_time_daily` per-session usage log.

| Column | Type |
|---|---|
| id | bigint PK |
| user_id | FK → `users.id` cascade delete nullable |
| device_id | string nullable |
| screen_name | string nullable |
| start_time | datetime nullable |
| end_time | datetime nullable |
| timestamps | |

!!! note
    New code writes to `screen_time_daily`. `screen_usages` is kept for historical data and the legacy activity-history view.

---

## Premium

### `subscriptions` *(2026-04-28)*

Subscription state, populated by the RevenueCat webhook. Source for the Premium conversion dashboard.

| Column | Type | Notes |
|---|---|---|
| id | bigint PK | |
| user_id | FK → `users.id` cascade delete | |
| provider | string(32) default `revenuecat` | |
| revenuecat_app_user_id | string(128) nullable | Equals `users.id` as a string. |
| entitlement | string(64) | e.g. `premium`. |
| is_active | boolean default `false` | |
| product_identifier | string(191) nullable | |
| period_type | string(32) nullable | `TRIAL` \| `NORMAL` \| `INTRO`. |
| store | string(32) nullable | `APP_STORE` \| `PLAY_STORE` \| … |
| environment | string(32) nullable | `SANDBOX` \| `PRODUCTION`. SANDBOX excluded from dashboards. |
| expires_at | timestamp nullable | |
| last_event_type | string(64) nullable | e.g. `INITIAL_PURCHASE`, `EXPIRATION`. |
| last_event_id | unsigned bigint nullable | |
| last_event_at | timestamp nullable | |
| last_event_payload | json nullable | |
| timestamps | | |
| **unique** | `(user_id, provider, entitlement)` | |
| **index** | `(provider, is_active)`, `expires_at` | |

---

## Push & notifications

### `firebase_tokens`

FCM device tokens used to push alerts to children or parents.

| Column | Type | Notes |
|---|---|---|
| id | bigint PK | |
| user_id | FK → `users.id` (cascade on delete since 2026-05-23) nullable | |
| token | longtext not null | FCM registration token. |
| device_id | longtext nullable | |
| status | enum (`Active`, `Inactive`) default `Active` | |
| created_at / updated_at | timestamp `useCurrent()` + `useCurrentOnUpdate()` | |
| softDeletes | | |

### `notifications`

Laravel's default UUID-keyed polymorphic notifications table (`notifiable_type`, `notifiable_id`, `data`, `read_at`).

---

## Content (admin-managed)

### `terms`, `policies`

Parallel tables: long-form text editable from the admin panel, shown in-app on the T&C / Privacy screens.

| Column | Type |
|---|---|
| id | bigint PK |
| description | longtext nullable (English) |
| descriptionSpa | longtext nullable (Spanish) |
| status | enum (`Active`, `Inactive`) default `Active` |
| timestamps | |

### `faqs`

Admin-managed FAQ entries (bilingual).

| Column | Type |
|---|---|
| id | bigint PK |
| question / questionSpa | text nullable |
| answer / answerSpa | text nullable |
| status | enum (`Active`, `Pending`, `Rejected`) default `Pending` |
| timestamps | |

### `learns`

Educational content for the in-app Learn module.

| Column | Type |
|---|---|
| id | bigint PK |
| description | longtext nullable |
| descriptionSpa | longtext nullable |
| timestamps | |

---

## Feedback

### `report_bugs`

In-app bug reports. Extended several times (2026-05 onward) with structured diagnostics.

| Column | Type | Notes |
|---|---|---|
| id | bigint PK | |
| user_id | FK → `users.id` cascade delete nullable | |
| child_user_id | FK → `users.id` nullable (indexed) | *(Added 2026-06-02)* Parent reporting on behalf of a child. |
| device_id | string nullable | |
| bug_type | string nullable | |
| description | text nullable | |
| image | text nullable | Legacy single-image URL. |
| images | json nullable | *(2026-05-13)* New multi-image shape. |
| reproducibility | enum (`always`, `sometimes`, `once`) | *(2026-05-12)* |
| affected_platforms | json nullable | *(2026-05-12)* |
| steps_to_reproduce | text nullable | *(2026-05-12)* |
| app_version, os_name, os_version | string nullable | *(2026-05-12)* |
| contact_back | boolean default `false` | *(2026-05-12)* |
| contact_email | string nullable | *(2026-05-12)* |
| diagnostics | json nullable | *(2026-05-12)* |
| source_app | string(32) nullable (indexed) | *(2026-06-02)* Which app filed it. |
| status | enum (`Submitted`, `In Progress`, `Solved`, `Closed`) | `On Progress` renamed to `In Progress` (2026-05-12). |
| timestamps | | |

### `report_bug_replies` *(2026-06-01)*

Admin email replies sent on a bug report (for threading).

| Column | Type | Notes |
|---|---|---|
| id | bigint PK | |
| report_bug_id | FK → `report_bugs.id` | |
| admin_id | FK → `users.id` | |
| message_id_header | string(255) (indexed) | Email Message-ID for threading. |
| subject | string | |
| sent_at | timestamp | |
| timestamps | | |

### `question_responses`

Onboarding-survey answers.

| Column | Type |
|---|---|
| id | bigint PK |
| user_id | FK → `users.id` cascade delete nullable |
| device_id | string nullable |
| spend_time | string nullable (time-spent estimate) |
| occupation | string nullable |
| hear_from | string nullable (discovery source) |
| havit | string nullable (current habit; key spelled `havit` by contract) |
| timestamps | |

!!! note
    The `reviews` table was dropped on 2026-05-13 and is no longer part of the schema. In-app rating now uses the native store review prompt, not a server-stored review.

---

## System

### `system_settings`

Admin-editable branding / contact configuration (single row).

| Column | Type |
|---|---|
| id | bigint PK |
| title, email, number, system_name, address, copyright_text, logo, favicon | string nullable |
| description | longtext nullable |
| timestamps + softDeletes | |

### `mail_settings`

Admin-editable SMTP settings (single row): `mailer`, `host`, `port`, `username`, `form_address`, `password`, `encryption`. Note transactional mail is driven by `.env` / `config/mail.php` (MailerSend), not this table.

### `cache`, `cache_locks`, `jobs`, `job_batches`, `failed_jobs`

Laravel defaults for the cache store and queue worker.
