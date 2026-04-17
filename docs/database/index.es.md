# Base de datos

El backend de Back To Life utiliza **MySQL 8.4** a través de Eloquent de Laravel. Los cambios de esquema se versionan mediante [migraciones](https://laravel.com/docs/migrations) en `backend/database/migrations/`.

## Secciones

- [Referencia de esquema](schema.md) — cada tabla, columna por columna.
- [Dominio de controles parentales](parental-controls.md) — cómo se relacionan `family_links`, `parental_controls`, `sleep_schedules` y `strict_shedules`.

## Mapa general

```
┌──────────────────────┐
│       users          │  role: Admin | User | Parent
└──────┬───────┬───────┘
       │       │
       │       │  (padre/hijo)
       │       ▼
       │  ┌─────────────┐        ┌──────────────────────┐
       │  │family_links │───1:1──│  parental_controls   │
       │  └──────┬──────┘        └──────────────────────┘
       │         │
       │         └──1:N──▶ ┌──────────────────┐
       │                   │  sleep_schedules │ (entre semana + fin de semana)
       │                   └──────────────────┘
       │
       ├──1:N──▶ ┌──────────────────┐
       │         │ screen_time_daily│  (uso por app, totales diarios)
       │         └──────────────────┘
       │
       ├──1:N──▶ ┌──────────────────┐
       │         │    activities    │  (registro de sesiones)
       │         └──────────────────┘
       │
       ├──1:1──▶ ┌──────────────────┐
       │         │  strict_shedules │  (ventana del filtro de redes sociales)
       │         └────────┬─────────┘
       │                  └──1:N──▶ ┌────────────┐
       │                            │ strict_data│  (auditoría por sesión)
       │                            └────────────┘
       │
       ├──1:N──▶ ┌──────────────────┐
       │         │  firebase_tokens │  (destino de push)
       │         └──────────────────┘
       │
       └──1:N──▶ activities · reviews · report_bugs · question_responses · app_statuses
```

## Agrupaciones por dominio

| Grupo | Tablas |
|---|---|
| **Identidad y autenticación** | `users`, `personal_access_tokens`, `password_reset_tokens`, `sessions` |
| **Controles parentales** | `family_links`, `parental_controls`, `sleep_schedules` |
| **Actividad y uso** | `activities`, `screen_time_daily`, `screen_usages` (legado), `strict_shedules`, `strict_data` |
| **Push y notificaciones** | `firebase_tokens`, `notifications` |
| **Contenido (gestionado por admin)** | `terms`, `policies`, `faqs`, `learns`, `app_statuses` |
| **Feedback** | `reviews`, `report_bugs`, `question_responses` |
| **Sistema** | `system_settings`, `mail_settings`, `cache`, `jobs` |

## Convenciones

- **Claves primarias**: `bigint unsigned auto_increment` vía `$table->id()`.
- **Timestamps**: toda tabla tiene `created_at` + `updated_at` salvo indicación contraria. Algunas añaden `softDeletes()` (`deleted_at`).
- **Claves foráneas** con `constrained()->cascadeOnDelete()`, para que al eliminar un `user` o `family_link` se limpien las filas dependientes.
- **Columnas JSON** se deserializan como arrays en PHP / listas en Dart a nivel de modelo (p. ej. `sleep_schedules.days`, `parental_controls.always_allowed_apps`).
- **Horas** (start_time / end_time) se guardan como `TIME` (`HH:MM:SS`); el cliente envía y lee `HH:MM`. Los controladores hacen la conversión.
- **Zonas horarias**: `strict_shedules.timezone` y `sleep_schedules.timezone` almacenan la zona IANA (p. ej. `Europe/Rome`) en que se interpretan los tiempos.

## Numeración de migraciones

Las migraciones llevan timestamp `YYYY_MM_DD_NNNNNN`. Las tres primeras (`0001_01_01_*`) son las de Laravel por defecto (users, cache, jobs). El trabajo desde enero de 2026 usa fechas reales.

Adiciones del dominio parental (desde marzo de 2026):

| Fecha | Migración |
|---|---|
| 2026-03-23 | `add_parent_role_to_users_table`, `create_family_links_table`, `create_parental_controls_table` |
| 2026-03-24 | `unique_email_per_role_in_users_table` |
| 2026-03-31 | `create_screen_time_daily_table` |
| 2026-04-16 | `add_weekday_weekend_limits_to_parental_controls_table`, `create_sleep_schedules_table`, `add_always_allowed_apps_to_parental_controls_table` |
| 2026-04-17 | `add_notification_preferences_to_parental_controls_table`, `add_battery_fields_to_users_table` |

## Ejecutar migraciones

```bash
# BD de desarrollo desde cero
php artisan migrate:fresh --seed

# Aplicar migraciones pendientes sobre una BD existente
php artisan migrate

# Deshacer el último lote
php artisan migrate:rollback
```

Al cambiar a una rama que añade migraciones, ejecuta siempre `php artisan migrate` antes de arrancar la app; si no, los endpoints que referencian columnas nuevas fallarán con "Unknown column".
