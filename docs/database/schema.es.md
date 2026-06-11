# Referencia de Esquema

Todas las tablas del backend, agrupadas por dominio, reflejando las migraciones tal como están aplicadas (no la forma original de creación). Los tipos de columna siguen el schema builder de Laravel (`$table->id()` produce `bigint unsigned auto_increment`, `$table->foreignId(...)` produce `bigint unsigned` con una restricción FK, etc.).

Salvo que se indique, cada tabla tiene `created_at` + `updated_at` (`timestamps()`).

!!! note
    La tabla obsoleta de fila única `strict_shedules` (typo intencional) se mantiene sincronizada desde las tablas canónicas `strict_schedule_blocks` + `strict_schedule_filters` para clientes antiguos. Los filtros nuevos se añaden solo a las tablas canónicas. Ver [Horario estricto (hijo)](#horario-estricto-hijo).

---

## Identidad y autenticación

### `users`

La única tabla de usuarios para todos los roles. Admin, User estándar (hijo) y Parent se diferencian por la columna `role`. El email es único por rol (la migración `unique_email_per_role_in_users_table`, 2026-03-24, relajó el unique global para que una misma dirección pueda registrarse una vez por rol, p.ej. padre + hijo).

| Columna | Tipo | Notas |
|---|---|---|
| id | bigint PK | |
| name | string nullable | |
| surname | string nullable | *(Añadido 2026-06-05)* |
| date_of_birth | date nullable | *(Añadido 2026-06-05)* |
| email | string nullable (único por rol) | |
| email_verified_at | timestamp nullable | |
| password | string nullable | Bcrypt. |
| avatar | text nullable | URL. |
| terms | boolean default `false` | T&C aceptados. |
| google_id | string nullable | *(Añadido 2026-01-24)* |
| is_admin_created | boolean default `false` | |
| device_id | string nullable | |
| language | string nullable | `en` \| `es` \| … |
| timezone | string(64) nullable | *(Añadido 2026-04-29)* Nombre IANA. |
| provider_id | string nullable (indexado) | Proveedor social (`google`, `apple`). |
| provider_user_id | string nullable (indexado) | Id externo del proveedor. |
| reset_code | string nullable | OTP de restablecimiento. |
| reset_code_expires_at | timestamp nullable | |
| role | enum (`Admin`, `User`, `Parent`) default `User` | |
| status | enum (`Pending`, `Rejected`, `Active`, `Deactivate`, `Banned`) default `Active` | |
| remember_token | string | |
| timestamps + softDeletes | | |

!!! note
    Los campos de batería (`battery_level`, `battery_charging`, `battery_updated_at`) estuvieron brevemente en `users` pero se movieron a [`devices`](#devices) el 2026-05-21. Ya no existen en `users`.

### `personal_access_tokens`

Almacén de tokens de Laravel Sanctum (migración por defecto de `laravel/sanctum`). Las apps móviles se autentican con estos bearer tokens.

### `password_reset_tokens`, `sessions`

Por defecto de Laravel. `sessions` respalda la autenticación por cookie de sesión usada por el panel admin.

---

## Familia y controles parentales

Ver [Dominio de Controles Parentales](parental-controls.md) para la guía conceptual.

### `family_links`

Una fila por hijo, tras el refactor multi-padre (2026-05-18). Los padres se vinculan a través de la tabla join `family_link_parents`, no por una columna en esta tabla.

| Columna | Tipo | Notas |
|---|---|---|
| id | bigint PK | |
| child_user_id | FK → `users.id` cascade delete (único) | Un link por hijo. |
| app_blocking_globally_enabled | boolean default `false` | Interruptor maestro del bloqueo de apps. |
| timestamps | | |

!!! note
    Las columnas originales `parent_user_id`, `status`, `invite_code`, `invite_expires_at` fueron eliminadas por el refactor (las invitaciones pasaron a `family_link_invites`, los padres a `family_link_parents`). Las columnas `passcode_hash` / `passcode_set_at` que estuvieron brevemente aquí pasaron a [`parent_passcodes`](#parent_passcodes) el 2026-06-02.

### `family_link_parents` *(2026-05-18)*

Tabla join que vincula padres a un `family_link` (varios padres por hijo).

| Columna | Tipo | Notas |
|---|---|---|
| family_link_id | FK → `family_links.id` cascade delete | Parte de la PK compuesta. |
| parent_user_id | FK → `users.id` cascade delete (indexado) | Parte de la PK compuesta. |
| added_at | timestamp `useCurrent()` | |
| **primary** | `(family_link_id, parent_user_id)` | Sin id surrogado ni timestamps. |

### `family_link_invites` *(2026-05-18)*

Invitaciones pendientes hijo-a-padre (código redimido por un padre).

| Columna | Tipo | Notas |
|---|---|---|
| id | bigint PK | |
| child_user_id | FK → `users.id` cascade delete | El hijo que invita. |
| code | string(32) (indexado) | |
| expires_at | timestamp (indexado) | Típicamente 24 h. |
| used_at | timestamp nullable | Null hasta redimirse. |
| used_by_parent_id | FK → `users.id` nullable | Padre que la redimió. |
| timestamps | | |

### `parental_controls`

Una fila por `family_link`. Contiene las restricciones configuradas por el padre para ese hijo.

| Columna | Tipo | Notas |
|---|---|---|
| id | bigint PK | |
| family_link_id | FK → `family_links.id` cascade delete | Efectivamente 1:1. |
| daily_screen_time_limit_minutes | int nullable | **Columna legacy de valor único**, mantenida por compatibilidad. |
| daily_limit_weekday_minutes | int nullable | *(Añadido 2026-04-16)* Tope diario Lun–Vie. |
| daily_limit_weekend_minutes | int nullable | *(Añadido 2026-04-16)* Tope diario Sáb–Dom. |
| always_allowed_apps | json nullable | *(Añadido 2026-04-16)* Array de `{ name, emoji?, package? }`. Permanecen desbloqueadas durante las ventanas de sueño. |
| notification_preferences | json nullable | *(Añadido 2026-04-17)* Toggles de notificación por hijo + horas de silencio. |
| timestamps | | |

!!! note
    `settings_locked` se eliminó el 2026-05-28; el estado de bloqueo ya no se almacena aquí.

### `parent_passcodes` *(2026-06-02)*

El propio código de la app del padre (uno por padre). Reemplaza las columnas `passcode_hash` que estuvieron brevemente en `family_links`.

| Columna | Tipo | Notas |
|---|---|---|
| id | bigint PK | |
| parent_user_id | FK → `users.id` | |
| passcode_hash | string(255) | PIN hasheado. |
| passcode_set_at | timestamp | |
| timestamps | | |

### `sleep_schedules` *(2026-04-16)*

Ventanas de "bloqueo de descanso" a nivel dispositivo. Hasta dos filas por hijo: una para noches entre semana, otra para noches de fin de semana.

| Columna | Tipo | Notas |
|---|---|---|
| id | bigint PK | |
| family_link_id | FK → `family_links.id` cascade delete | |
| bucket | string (`weeknight` \| `weekend`) | |
| enabled | boolean default `true` | |
| start_time | time | `HH:MM:SS` en la zona horaria almacenada. |
| end_time | time | Si `< start_time` la ventana cruza medianoche. |
| days | json | Array de ints de día ISO (1 = Lun … 7 = Dom). |
| timezone | string nullable | Nombre IANA. |
| timestamps | | |
| **unique** | `(family_link_id, bucket)` | Máx 2 filas por hijo. |

---

## Dispositivos y bloqueo de apps

### `devices` *(2026-05-18)*

Una fila por dispositivo registrado por usuario. Lleva la telemetría de batería (movida aquí desde `users` el 2026-05-21).

| Columna | Tipo | Notas |
|---|---|---|
| id | bigint PK | |
| user_id | FK → `users.id` cascade delete | |
| device_identifier | string | Id estable por instalación. |
| platform | enum (`ios`, `android`) | |
| os_version | string nullable | |
| app_version | string nullable | |
| last_seen_at | timestamp nullable (indexado) | Señal "última vez visto" de la vista de Retención. |
| battery_level | unsigned tinyint nullable | 0–100. |
| battery_charging | boolean nullable | |
| battery_updated_at | timestamp nullable | |
| timestamps | | |
| **unique** | `(user_id, device_identifier)` | |

### `device_app_blocking` *(2026-05-18)*

Estado de bloqueo + estado de configuración por dispositivo.

| Columna | Tipo | Notas |
|---|---|---|
| id | bigint PK | |
| device_id | FK → `devices.id` cascade delete (único) | Una fila por dispositivo. |
| state | enum | Estado del ciclo de bloqueo. |
| platform | enum (`ios`, `android`) | |
| last_synced_at | timestamp nullable | |
| setup_completed_at | timestamp nullable | |
| timestamps | | |

### `device_managed_apps` *(2026-05-18)*

Las apps bajo gestión en un dispositivo. Las columnas `*_shielded` se renombraron a `*_blocked` el 2026-05-21.

| Columna | Tipo | Notas |
|---|---|---|
| id | bigint PK | |
| device_id | FK → `devices.id` cascade delete | |
| identifier | string | Token de label iOS o package Android. |
| type | enum (`ios_label`, `android_package`) | |
| app_label | string | |
| icon_url | string nullable | |
| parent_blocked | boolean default `false` (indexado) | Fijado por el padre. |
| parent_blocked_updated_by | FK → `users.id` nullable | |
| parent_blocked_updated_at | timestamp nullable | |
| self_blocked | boolean default `false` (indexado) | Fijado por el hijo (autocontrol). |
| self_blocked_updated_at | timestamp nullable | |
| last_seen_at | timestamp nullable | |
| last_verified_at | timestamp nullable (indexado) | *(Añadido 2026-05-19)* |
| timestamps | | |
| **unique** | `(device_id, identifier)` | |

### `device_unblock_requests` *(2026-05-18)*

Solicitud de un hijo para desbloquear temporalmente una app gestionada, a la espera de aprobación del padre.

| Columna | Tipo | Notas |
|---|---|---|
| id | bigint PK | |
| device_id | FK → `devices.id` cascade delete | |
| managed_app_id | FK → `device_managed_apps.id` cascade delete | |
| child_user_id | FK → `users.id` cascade delete | |
| duration_seconds | unsigned int | Duración de desbloqueo solicitada. |
| state | enum (`pending`, `approved`, `denied`, `expired`) | |
| requested_at | timestamp `useCurrent()` | |
| resolved_at | timestamp nullable | |
| resolved_by_parent_id | FK → `users.id` nullable | |
| timestamps | | |

### `device_managed_app_events` *(2026-05-19)*

Registro append-only de acciones de bloqueo/desbloqueo sobre apps gestionadas.

| Columna | Tipo | Notas |
|---|---|---|
| id | bigint PK | |
| device_id | FK → `devices.id` | |
| identifier | string | |
| app_label | string | |
| action | enum | Bloquear / desbloquear / etc. |
| reason | string(255) nullable | |
| actor_user_id | FK → `users.id` nullable | Padre o hijo que actuó. |
| created_at | timestamp `useCurrent()` | Sin `updated_at`. |

---

## Horario estricto (hijo)

Los datos canónicos del horario estricto viven en dos tablas (`strict_schedule_blocks` + `strict_schedule_filters`). La tabla legacy de fila única `strict_shedules` es un espejo obsoleto mantenido sincronizado para clientes antiguos.

### `strict_schedule_blocks` *(2026-04-28)*

Ventanas de horario por tipo (entre semana + fin de semana), una fila cada una.

| Columna | Tipo | Notas |
|---|---|---|
| id | bigint PK | |
| user_id | FK → `users.id` cascade delete | |
| kind | string(16) | `weekday` \| `weekend`. |
| is_active | boolean default `false` | |
| start_time | time nullable | |
| end_time | time nullable | Puede cruzar medianoche. |
| days | json nullable | |
| timezone | string(64) nullable | |
| last_start_notified_date | date nullable | De-duplica notificaciones push. |
| last_end_notified_date | date nullable | |
| timestamps | | |
| **unique** | `(user_id, kind)` | |
| **index** | `(user_id, is_active)` | |

### `strict_schedule_filters` *(2026-04-28)*

Flags de filtro de contenido por usuario. Una fila por usuario. Todas las columnas boolean `NOT NULL DEFAULT false`. Es el hogar canónico de cada filtro; los filtros nuevos se añaden solo aquí.

| Grupo de columnas | Columnas |
|---|---|
| YouTube | `youtube_shorts`, `youtube_comments`, `youtube_subscriptions` *(2026-04-30)*, `youtube_home` *(2026-04-30)* |
| Instagram | `instagram_reels`, `instagram_explore`, `instagram_hide_explore` *(2026-05-11)*, `instagram_hide_suggestions` *(2026-05-11)*, `instagram_likeComments`, `instagram_story`, `instagramDmOnly` |
| Twitter / X *(2026-05-11)* | `twitter_for_you`, `twitter_following`, `twitter_grok`, `twitter_notifications`, `twitter_messages` |
| TikTok *(2026-06-09)* | `tiktok_for_you`, `tiktok_dm_only`, `tiktok_search_only`, `tiktok_notifications` |

Más `id`, `user_id` (FK, único), `timezone` (string(64) nullable), `timestamps`.

### `strict_streak_days` *(2026-04-29)*

Una fila por hijo por día, registrando si el horario estaba programado y si se respetó. Respalda el contador de racha.

| Columna | Tipo | Notas |
|---|---|---|
| id | bigint PK | |
| user_id | FK → `users.id` cascade delete | |
| date | date | |
| was_scheduled | boolean | Si había un bloque activo ese día. |
| was_honored | boolean | Si el hijo lo mantuvo activo. |
| timestamps | | |
| **unique** | `(user_id, date)` | |
| **index** | `(user_id, date, was_scheduled)` | |

### `strict_shedules` *(espejo legacy)*

Filtro + ventana obsoleto de fila única por hijo. **Nombrado `strict_shedules` (falta una c) intencionalmente — no "arreglar" el typo sin una migración de renombrado.** Mantenido sincronizado desde las tablas canónicas para clientes antiguos. Los filtros de Twitter / TikTok NO se reflejan aquí deliberadamente (ningún cliente legacy los expone).

| Columna | Tipo | Notas |
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
    `instagram_autoplay_off` se eliminó de esta tabla el 2026-05-14; `instagram_hide_explore` / `instagram_hide_suggestions` también se añadieron y luego se eliminaron. Autoplay-off es ahora una preferencia por app a nivel dispositivo, no un filtro de horario.

### `strict_data`

Rastro de auditoría a nivel sesión de cuándo cada horario estricto (legacy) se activó / desactivó.

| Columna | Tipo |
|---|---|
| id | bigint PK |
| strict_id | FK → `strict_shedules.id` cascade delete |
| started_at | datetime nullable |
| disabled_at | datetime nullable |
| timestamps | |

### `app_statuses`

Snapshot por hijo de los toggles del filtro de redes sociales, independiente de la ventana de horario. Refleja los siete booleanos de Instagram/YouTube de `strict_shedules`.

| Columna | Tipo |
|---|---|
| id | bigint PK |
| user_id | FK → `users.id` cascade delete nullable |
| device_id | string nullable |
| youtube_shorts, youtube_comments | boolean nullable |
| instagram_reels, instagram_explore, instagram_likeComments, instagram_story, instagramDmOnly | boolean nullable |
| timestamps | |

---

## Horario estricto (gestionado por el padre)

Horarios fijados por el padre para un hijo, espejando las tablas del hijo. Escritos por la app parental vía `PUT /parent/children/{id}/schedule`.

### `parent_strict_schedule_blocks` *(2026-05-28)*

Forma idéntica a [`strict_schedule_blocks`](#strict_schedule_blocks-2026-04-28).

### `parent_strict_schedule_filters` *(2026-05-28)*

Forma idéntica a [`strict_schedule_filters`](#strict_schedule_filters-2026-04-28), incluyendo las columnas de YouTube/Instagram/Twitter en su creación y las columnas `tiktok_*` añadidas el 2026-06-09.

---

## Actividad y uso

### `activities`

Registro de login/logout a nivel sesión por usuario. `login_time` es la señal de "usuario activo" usada por la vista de Retención.

| Columna | Tipo |
|---|---|
| id | bigint PK |
| user_id | FK → `users.id` cascade delete |
| device_id | string nullable |
| login_time | timestamp nullable |
| logout_time | timestamp nullable |
| total_time | string nullable (HH:MM:SS) |
| timestamps | |

### `screen_time_daily` *(2026-03-31)*

Totales por app y por día reportados por la app del hijo. Es lo que leen el gráfico de barras del panel parental y la analítica in-app. Reemplaza la tabla anterior `screen_usages`.

| Columna | Tipo | Notas |
|---|---|---|
| id | bigint PK | |
| user_id | FK → `users.id` cascade delete | |
| date | date | Fecha local en la zona horaria del hijo (provista por el cliente). |
| timezone | string(64) | Nombre IANA. |
| app | string(32) | Uno de `youtube` \| `instagram` \| `twitter` \| `tiktok`. |
| source | string(16) | `webview` (dentro de BackToLife) \| `native` (UsageStats / DeviceActivity). |
| seconds | unsigned int default 0 | Total del día. |
| updated_at | timestamp | `useCurrent()->useCurrentOnUpdate()`. Sin `created_at`. |
| **unique** | `(user_id, date, app, source)` | Clave de upsert. |
| **index** | `(user_id, date)` | |

### `screen_usages` *(legacy)*

Registro de uso por sesión anterior a `screen_time_daily`.

| Columna | Tipo |
|---|---|
| id | bigint PK |
| user_id | FK → `users.id` cascade delete nullable |
| device_id | string nullable |
| screen_name | string nullable |
| start_time | datetime nullable |
| end_time | datetime nullable |
| timestamps | |

!!! note
    El código nuevo escribe en `screen_time_daily`. `screen_usages` se conserva por datos históricos y la vista legacy de historial de actividad.

---

## Premium

### `subscriptions` *(2026-04-28)*

Estado de suscripción, poblado por el webhook de RevenueCat. Fuente del panel de conversión Premium.

| Columna | Tipo | Notas |
|---|---|---|
| id | bigint PK | |
| user_id | FK → `users.id` cascade delete | |
| provider | string(32) default `revenuecat` | |
| revenuecat_app_user_id | string(128) nullable | Igual a `users.id` como string. |
| entitlement | string(64) | p.ej. `premium`. |
| is_active | boolean default `false` | |
| product_identifier | string(191) nullable | |
| period_type | string(32) nullable | `TRIAL` \| `NORMAL` \| `INTRO`. |
| store | string(32) nullable | `APP_STORE` \| `PLAY_STORE` \| … |
| environment | string(32) nullable | `SANDBOX` \| `PRODUCTION`. SANDBOX excluido de los paneles. |
| expires_at | timestamp nullable | |
| last_event_type | string(64) nullable | p.ej. `INITIAL_PURCHASE`, `EXPIRATION`. |
| last_event_id | unsigned bigint nullable | |
| last_event_at | timestamp nullable | |
| last_event_payload | json nullable | |
| timestamps | | |
| **unique** | `(user_id, provider, entitlement)` | |
| **index** | `(provider, is_active)`, `expires_at` | |

---

## Push y notificaciones

### `firebase_tokens`

Tokens FCM de dispositivo usados para enviar alertas a hijos o padres.

| Columna | Tipo | Notas |
|---|---|---|
| id | bigint PK | |
| user_id | FK → `users.id` (cascade on delete desde 2026-05-23) nullable | |
| token | longtext not null | Token de registro FCM. |
| device_id | longtext nullable | |
| status | enum (`Active`, `Inactive`) default `Active` | |
| created_at / updated_at | timestamp `useCurrent()` + `useCurrentOnUpdate()` | |
| softDeletes | | |

### `notifications`

Tabla polimórfica de notificaciones por defecto de Laravel, con id UUID (`notifiable_type`, `notifiable_id`, `data`, `read_at`).

---

## Contenido (gestionado por admin)

### `terms`, `policies`

Tablas paralelas: texto largo editable desde el panel admin, mostrado in-app en las pantallas de T&C / Privacidad.

| Columna | Tipo |
|---|---|
| id | bigint PK |
| description | longtext nullable (inglés) |
| descriptionSpa | longtext nullable (español) |
| status | enum (`Active`, `Inactive`) default `Active` |
| timestamps | |

### `faqs`

Entradas FAQ gestionadas por admin (bilingües).

| Columna | Tipo |
|---|---|
| id | bigint PK |
| question / questionSpa | text nullable |
| answer / answerSpa | text nullable |
| status | enum (`Active`, `Pending`, `Rejected`) default `Pending` |
| timestamps | |

### `learns`

Contenido educativo del módulo Learn in-app.

| Columna | Tipo |
|---|---|
| id | bigint PK |
| description | longtext nullable |
| descriptionSpa | longtext nullable |
| timestamps | |

---

## Feedback

### `report_bugs`

Reportes de bugs in-app. Extendido varias veces (desde 2026-05) con diagnósticos estructurados.

| Columna | Tipo | Notas |
|---|---|---|
| id | bigint PK | |
| user_id | FK → `users.id` cascade delete nullable | |
| child_user_id | FK → `users.id` nullable (indexado) | *(Añadido 2026-06-02)* Padre reportando en nombre de un hijo. |
| device_id | string nullable | |
| bug_type | string nullable | |
| description | text nullable | |
| image | text nullable | URL de imagen única (legacy). |
| images | json nullable | *(2026-05-13)* Nueva forma multi-imagen. |
| reproducibility | enum (`always`, `sometimes`, `once`) | *(2026-05-12)* |
| affected_platforms | json nullable | *(2026-05-12)* |
| steps_to_reproduce | text nullable | *(2026-05-12)* |
| app_version, os_name, os_version | string nullable | *(2026-05-12)* |
| contact_back | boolean default `false` | *(2026-05-12)* |
| contact_email | string nullable | *(2026-05-12)* |
| diagnostics | json nullable | *(2026-05-12)* |
| source_app | string(32) nullable (indexado) | *(2026-06-02)* Qué app lo registró. |
| status | enum (`Submitted`, `In Progress`, `Solved`, `Closed`) | `On Progress` renombrado a `In Progress` (2026-05-12). |
| timestamps | | |

### `report_bug_replies` *(2026-06-01)*

Respuestas de email del admin enviadas sobre un reporte de bug (para threading).

| Columna | Tipo | Notas |
|---|---|---|
| id | bigint PK | |
| report_bug_id | FK → `report_bugs.id` | |
| admin_id | FK → `users.id` | |
| message_id_header | string(255) (indexado) | Message-ID de email para threading. |
| subject | string | |
| sent_at | timestamp | |
| timestamps | | |

### `question_responses`

Respuestas de la encuesta de onboarding.

| Columna | Tipo |
|---|---|
| id | bigint PK |
| user_id | FK → `users.id` cascade delete nullable |
| device_id | string nullable |
| spend_time | string nullable (estimación de tiempo) |
| occupation | string nullable |
| hear_from | string nullable (fuente de descubrimiento) |
| havit | string nullable (hábito actual; clave escrita `havit` por contrato) |
| timestamps | |

!!! note
    La tabla `reviews` se eliminó el 2026-05-13 y ya no forma parte del esquema. La valoración in-app ahora usa el prompt nativo de reseña de la tienda, no una reseña almacenada en el servidor.

---

## Sistema

### `system_settings`

Configuración de marca / contacto editable por admin (fila única).

| Columna | Tipo |
|---|---|
| id | bigint PK |
| title, email, number, system_name, address, copyright_text, logo, favicon | string nullable |
| description | longtext nullable |
| timestamps + softDeletes | |

### `mail_settings`

Ajustes SMTP editables por admin (fila única): `mailer`, `host`, `port`, `username`, `form_address`, `password`, `encryption`. El correo transaccional se gobierna por `.env` / `config/mail.php` (MailerSend), no por esta tabla.

### `cache`, `cache_locks`, `jobs`, `job_batches`, `failed_jobs`

Por defecto de Laravel para el almacén de cache y el worker de colas.
