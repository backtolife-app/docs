# Referencia de esquema

Todas las tablas del backend, agrupadas por dominio. Los tipos siguen el schema builder de Laravel (`$table->id()` → `bigint unsigned auto_increment`, `$table->foreignId(...)` → `bigint unsigned` con FK, etc.).

Salvo indicación contraria, toda tabla lleva `created_at` + `updated_at` (`timestamps()`).

---

## Identidad y autenticación

### `users`

Tabla única para todos los roles — `Admin`, `User` (hijo) y `Parent` se diferencian por la columna `role`. El email era globalmente único; la migración `unique_email_per_role_in_users_table` (2026-03-24) lo relajó a único por `(email, role)`, así un usuario puede registrarse una vez por rol (padre + hijo con el mismo correo).

| Columna | Tipo | Notas |
|---|---|---|
| id | bigint PK | |
| name | string nullable | |
| email | string nullable (único, por rol) | |
| email_verified_at | timestamp nullable | |
| password | string nullable | Bcrypt. |
| avatar | text nullable | URL. |
| terms | boolean default `false` | T&C aceptados. |
| is_admin_created | boolean default `false` | |
| device_id | string nullable | Usado antes para fijar dispositivo. |
| language | string nullable | `en` \| `es` \| … |
| provider_id | string nullable (índice) | Proveedor social (`google`, `apple`). |
| provider_user_id | string nullable (índice) | Id del usuario en el proveedor. |
| google_id | string nullable | Añadido por `add_google_id_in_users_table`. |
| reset_code | string nullable | OTP de recuperación. |
| reset_code_expires_at | timestamp nullable | |
| role | enum (`Admin`, `User`, `Parent`) default `User` | |
| status | enum (`Pending`, `Rejected`, `Active`, `Deactivate`, `Banned`) default `Active` | |
| remember_token | string | |
| timestamps + softDeletes | | |

### `personal_access_tokens`

Store de tokens de Sanctum — provisto por la migración por defecto del paquete `laravel/sanctum`.

### `password_reset_tokens`, `sessions`

Defaults de Laravel. `sessions` solo se usa si se activa autenticación por sesión en el panel de admin.

---

## Controles parentales

Ver [Dominio de controles parentales](parental-controls.md) para la guía conceptual.

### `family_links`

Enlaza un padre con un hijo. Diseñado many-to-many — un hijo puede tener varios padres (custodia compartida), un padre puede enlazar varios hijos (hasta 10 por regla de negocio).

| Columna | Tipo | Notas |
|---|---|---|
| id | bigint PK | |
| parent_user_id | FK → `users.id` nullable, cascade delete | |
| child_user_id | FK → `users.id` nullable, cascade delete | |
| status | enum (`pending`, `active`, `revoked`) default `pending` | |
| invite_code | string único nullable | Código de 8 caracteres generado por `/child/invite`. |
| invite_expires_at | timestamp nullable | Normalmente 24 h. |
| timestamps | | |

### `parental_controls`

Una fila por `family_link` activo. Guarda las restricciones que el padre configura para ese hijo.

| Columna | Tipo | Notas |
|---|---|---|
| id | bigint PK | |
| family_link_id | FK → `family_links.id` cascade delete | Efectivamente 1:1. |
| settings_locked | boolean default `false` | Si true, el hijo no puede editar horario/actividades en su propia app. |
| daily_screen_time_limit_minutes | int nullable | **Columna de valor único legada**, conservada por retrocompatibilidad. |
| daily_limit_weekday_minutes | int nullable | *(Añadida 2026-04-16)* Tope diario L–V. |
| daily_limit_weekend_minutes | int nullable | *(Añadida 2026-04-16)* Tope diario S–D. |
| always_allowed_apps | json nullable | *(Añadida 2026-04-16)* Array de `{ name, emoji?, package? }`. Apps que siguen disponibles durante ventanas de sueño. |
| timestamps | | |

### `sleep_schedules` *(añadida 2026-04-16)*

Ventanas de "bloqueo de dispositivo" a nivel global. Hasta dos filas por hijo: una de entre semana, otra de fin de semana. Se aplicará desde la app hijo (pendiente — ver [PLAN.md](../../../parent/PLAN.md)).

| Columna | Tipo | Notas |
|---|---|---|
| id | bigint PK | |
| family_link_id | FK → `family_links.id` cascade delete | |
| bucket | string (`weeknight` \| `weekend`) | |
| enabled | boolean default `true` | |
| start_time | time | `HH:MM:SS` en la zona almacenada. |
| end_time | time | `HH:MM:SS`. Si `< start_time`, cruza la medianoche. |
| days | json | Array de enteros ISO-8601 (1 = Lunes … 7 = Domingo). |
| timezone | string nullable | Nombre IANA, p. ej. `Europe/Rome`. |
| timestamps | | |
| **único** | `(family_link_id, bucket)` | Máximo 2 filas por hijo. |

---

## Actividad y uso

### `screen_time_daily` *(2026-03-31)*

Totales por app y por día que reporta la app hijo. Fuente del gráfico de barras del dashboard del padre. Reemplaza a `screen_usages`.

| Columna | Tipo | Notas |
|---|---|---|
| id | bigint PK | |
| user_id | FK → `users.id` cascade delete | Usuario hijo. |
| date | date | Fecha local en la zona del hijo (enviada por cliente). |
| timezone | string(64) | Nombre IANA. |
| app | string(32) | Hoy: `youtube` \| `instagram`. |
| source | string(16) | `webview` (dentro de BackToLife) \| `native` (UsageStats). |
| seconds | unsigned int default 0 | Total del día. |
| updated_at | timestamp | Se refresca con `useCurrent()->useCurrentOnUpdate()`. Sin `created_at`. |
| **único** | `(user_id, date, app, source)` | Clave de upsert. |
| **índice** | `(user_id, date)` | Para consultas por rango de fechas. |

### `activities`

Registro de login/logout a nivel de sesión.

| Columna | Tipo |
|---|---|
| id | bigint PK |
| user_id | FK → `users.id` cascade delete |
| device_id | string nullable |
| login_time | timestamp nullable |
| logout_time | timestamp nullable |
| total_time | string nullable (HH:MM:SS) |
| timestamps | |

### `strict_shedules`

Filtro de redes sociales por hijo. **La tabla se llama `strict_shedules` (sin la "c") de forma intencional — no "arreglar" el typo sin una migración de renombrado.** Guarda qué funcionalidades de YouTube / Instagram ocultar y en qué ventana horaria. Una fila por hijo mediante upsert sobre `user_id`.

| Columna | Tipo | Notas |
|---|---|---|
| id | bigint PK | |
| user_id | FK → `users.id` cascade delete nullable | Id del hijo. |
| device_id | string nullable | |
| timezone | string nullable | |
| youtube_shorts | boolean nullable | |
| youtube_comments | boolean nullable | |
| instagram_reels | boolean nullable | |
| instagram_explore | boolean nullable | |
| instagram_likeComments | boolean nullable | |
| instagram_story | boolean nullable | |
| instagramDmOnly | boolean nullable | |
| start_time | time nullable | Inicio de ventana (TZ servidor). |
| end_time | time nullable | Fin de ventana (puede ser al día siguiente). |
| days | json nullable | Días en los que aplica. |
| is_active | boolean nullable | |
| last_start_notified_date | date nullable | Evita duplicar notificaciones push. |
| last_end_notified_date | date nullable | |
| timestamps | | |

### `strict_data`

Historial de cuándo se activó / desactivó cada horario estricto. Sirve para calcular rachas.

| Columna | Tipo |
|---|---|
| id | bigint PK |
| strict_id | FK → `strict_shedules.id` cascade delete |
| started_at | datetime nullable |
| disabled_at | datetime nullable |
| timestamps | |

### `screen_usages` *(legado)*

Log previo a `screen_time_daily` — por sesión en vez de por día.

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
    El código nuevo escribe en `screen_time_daily`. `screen_usages` se mantiene por los datos históricos y el endpoint de historial de actividad.

### `app_statuses`

Estado por defecto de los toggles de filtro de redes sociales por hijo. Refleja las siete columnas booleanas de `strict_shedules` pero sin ventana horaria — una instantánea de "qué está activado ahora" independiente del horario.

---

## Push y notificaciones

### `firebase_tokens`

Tokens FCM usados para enviar alertas a hijos o padres.

| Columna | Tipo | Notas |
|---|---|---|
| id | bigint PK | |
| user_id | FK → `users.id` (restrict en delete, cascade en update) nullable | |
| token | longtext not null | Token de registro FCM. |
| device_id | longtext nullable | |
| status | enum (`Active`, `Inactive`) default `Active` | |
| created_at / updated_at | timestamp auto | `useCurrent()` + `useCurrentOnUpdate()` |
| softDeletes | | |

!!! warning
    El servicio de envío (padre → hijo con comando "bloquear ahora") **aún no está construido**. Hoy solo existen los endpoints de CRUD de tokens.

### `notifications`

Tabla polimórfica de notificaciones por defecto de Laravel (`notifiable_type`, `notifiable_id`). Actualmente se usa solo para alertas in-app.

---

## Contenido (gestionado por admin)

### `terms`, `policies`

Tablas paralelas — texto largo editable desde el panel de admin, mostrado en la app en T&C / Política.

| Columna | Tipo |
|---|---|
| id | bigint PK |
| description | longtext nullable (inglés) |
| descriptionSpa | longtext nullable (español) |
| status | enum (`Active`, `Inactive`) default `Active` |
| timestamps | |

### `faqs`

Preguntas frecuentes gestionadas por admin. Forma análoga a `terms` / `policies` más una columna `question`.

### `learns`

Contenido educativo (módulo de aprendizaje de Back To Life).

| Columna | Tipo |
|---|---|
| id | bigint PK |
| description | longtext nullable |
| descriptionSpa | longtext nullable |
| timestamps | |

---

## Feedback

### `reviews`

Valoraciones in-app.

| Columna | Tipo |
|---|---|
| id | bigint PK |
| user_id | FK → `users.id` cascade delete nullable |
| device_id | string nullable |
| rating | int nullable |
| comment | text nullable |
| reply | text nullable |
| status | enum (`Active`, `Inactive`) default `Active` |
| timestamps | |

### `report_bugs`

Reportes de bugs desde la app.

| Columna | Tipo |
|---|---|
| id | bigint PK |
| user_id | FK → `users.id` cascade delete nullable |
| device_id | string nullable |
| bug_type | string nullable |
| description | text nullable |
| image | text nullable (URL) |
| status | enum (`Submitted`, `On Progress`, `Solved`, `Closed`) |
| timestamps | |

### `question_responses`

Respuestas de la encuesta de onboarding: tiempo estimado, ocupación, cómo nos conoció, hábito actual.

---

## Sistema

### `system_settings`, `mail_settings`

Configuración editable por admin indexada por nombre. Comparten una forma `KeyValue`.

### `cache`, `jobs`

Defaults de Laravel para el store de caché y la cola.
