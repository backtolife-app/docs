# Database

The Back To Life backend runs on **MySQL 8.4** via Laravel Eloquent. Schema changes are version-controlled as [Laravel migrations](https://laravel.com/docs/migrations) under `backend/database/migrations/`.

## Sections

- [Schema Reference](schema.md) — every table, column by column.
- [Parental Controls Domain](parental-controls.md) — how `family_links`, `parental_controls`, `sleep_schedules`, and `strict_shedules` fit together.

## High-level map

```
┌──────────────────────┐
│       users          │  role: Admin | User | Parent
└──────┬───────┬───────┘
       │       │
       │       │  (parent/child)
       │       ▼
       │  ┌─────────────┐        ┌──────────────────────┐
       │  │family_links │───1:1──│  parental_controls   │
       │  └──────┬──────┘        └──────────────────────┘
       │         │
       │         └──1:N──▶ ┌──────────────────┐
       │                   │  sleep_schedules │ (weeknight + weekend)
       │                   └──────────────────┘
       │
       ├──1:N──▶ ┌──────────────────┐
       │         │ screen_time_daily│  (per-app usage, daily totals)
       │         └──────────────────┘
       │
       ├──1:N──▶ ┌──────────────────┐
       │         │    activities    │  (session log)
       │         └──────────────────┘
       │
       ├──1:1──▶ ┌──────────────────┐
       │         │  strict_shedules │  (social-media filter window)
       │         └────────┬─────────┘
       │                  └──1:N──▶ ┌────────────┐
       │                            │ strict_data│  (session-level audit)
       │                            └────────────┘
       │
       ├──1:N──▶ ┌──────────────────┐
       │         │  firebase_tokens │  (push target)
       │         └──────────────────┘
       │
       └──1:N──▶ activities · reviews · report_bugs · question_responses · app_statuses
```

## Domain groupings

| Group | Tables |
|---|---|
| **Identity & auth** | `users`, `personal_access_tokens`, `password_reset_tokens`, `sessions` |
| **Parental controls** | `family_links`, `parental_controls`, `sleep_schedules` |
| **Activity & usage** | `activities`, `screen_time_daily`, `screen_usages` (legacy), `strict_shedules`, `strict_data` |
| **Push & notifications** | `firebase_tokens`, `notifications` |
| **Content (admin-managed)** | `terms`, `policies`, `faqs`, `learns`, `app_statuses` |
| **Feedback** | `reviews`, `report_bugs`, `question_responses` |
| **System** | `system_settings`, `mail_settings`, `cache`, `jobs` |

## Conventions

- **Primary keys** are `bigint unsigned auto_increment` via `$table->id()`.
- **Timestamps**: every table has `created_at` + `updated_at` unless noted. Some tables additionally have `softDeletes()` (`deleted_at`).
- **Foreign keys** use `constrained()->cascadeOnDelete()` so removing a `user` or a `family_link` cleans up the dependent rows.
- **JSON columns** are cast to PHP arrays / Dart lists at the model layer (e.g. `sleep_schedules.days`, `parental_controls.always_allowed_apps`).
- **Times of day** (start_time / end_time) are stored in `TIME` format (`HH:MM:SS`) but the client sends and reads `HH:MM`. Controllers handle the format conversion.
- **Timezones**: `strict_shedules.timezone` and `sleep_schedules.timezone` store the IANA zone (e.g. `Europe/Rome`) the times are interpreted in.

## Migration numbering

Migrations are timestamped `YYYY_MM_DD_NNNNNN`. The first three migrations (`0001_01_01_*`) are Laravel defaults (users, cache, jobs). Feature work from Jan 2026 onwards uses real dates.

Parental-control additions (from March 2026 onwards):

| Date | Migration |
|---|---|
| 2026-03-23 | `add_parent_role_to_users_table`, `create_family_links_table`, `create_parental_controls_table` |
| 2026-03-24 | `unique_email_per_role_in_users_table` |
| 2026-03-31 | `create_screen_time_daily_table` |
| 2026-04-16 | `add_weekday_weekend_limits_to_parental_controls_table`, `create_sleep_schedules_table`, `add_always_allowed_apps_to_parental_controls_table` |

## Running migrations

```bash
# Fresh dev DB
php artisan migrate:fresh --seed

# Apply pending migrations in an existing DB
php artisan migrate

# Roll back the last batch
php artisan migrate:rollback
```

When pulling a branch that adds migrations, always run `php artisan migrate` before starting the app — otherwise API endpoints referencing new columns will crash with "Unknown column".
