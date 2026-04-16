# Dominio de controles parentales

El lado parental del esquema tiene cuatro tablas fáciles de confundir. Esta página explica para qué sirve cada una, quién escribe en ellas y cómo se relacionan.

## Las cuatro tablas

```
family_links ─── 1:1 ───▶ parental_controls
     │
     └─── 1:N ──▶ sleep_schedules        (bloqueo a nivel de dispositivo)

users ─── 1:1 ──▶ strict_shedules        (filtro de redes sociales)
```

| Tabla | Alcance | Forma | Quién escribe |
|---|---|---|---|
| `family_links` | Enlaza un padre ↔ un hijo. Many-to-many en global. | Una fila por enlace. | Padre + hijo (flujo de vinculación). |
| `parental_controls` | Ajustes por hijo (bloqueo, topes de pantalla, lista permitida). | Una fila por enlace. | App padre. |
| `sleep_schedules` | Bloqueo **a nivel dispositivo** durante horas de sueño. | Hasta **dos** filas por enlace (`bucket = weeknight | weekend`). | App padre. |
| `strict_shedules` | Filtro de funcionalidades de **redes sociales** (toggles de YouTube/Instagram durante una ventana). | Una fila por hijo. | App hijo (auto-configurada) o app padre (override). |

Modelo mental simple: **`strict_shedules` controla lo que se ve dentro de Instagram y YouTube. `sleep_schedules` controla si el móvil del hijo funciona o no.** No comparten datos ni código de aplicación.

---

## Vincular un hijo

1. **Hijo** llama `POST /child/invite`. El backend genera un `invite_code` de 8 caracteres, crea una fila en `family_links` con `status = pending` y `invite_expires_at = now + 24h`, y devuelve el código.
2. **Padre** escanea el QR (o escribe el código) → `POST /parent/children/link` con ese código.
3. El backend asigna `parent_user_id`, pasa `status` a `active`, crea la fila de `parental_controls` y limpia `invite_code`.
4. A partir de aquí el padre puede consultar el dashboard del hijo y enviar PUT de controles.

Desvincular deja `status = revoked` y no hace cascada — las filas de control y horario se conservan por auditoría. Un nuevo enlace crea una fila nueva; el histórico se preserva.

---

## Topes de tiempo de pantalla

`parental_controls` tiene tres columnas relevantes:

- `daily_screen_time_limit_minutes` — valor único **legado**.
- `daily_limit_weekday_minutes` — tope L–V.
- `daily_limit_weekend_minutes` — tope S–D.

La app padre siempre lee y escribe el par **entre semana + fin de semana**. La columna legada solo se consulta como fallback cuando las dos nuevas son nulas (p. ej. datos escritos por un cliente antiguo).

Lógica en cliente (`ParentalControl.todayLimitMinutes` en `children_model.dart`):

```dart
int? get todayLimitMinutes {
  final isWeekend = DateTime.now().weekday >= DateTime.saturday;
  return (isWeekend ? dailyLimitWeekendMinutes : dailyLimitWeekdayMinutes)
      ?? dailyScreenTimeLimitMinutes;
}
```

Se espera que la app hijo replique esa lógica al aplicar el límite. El backend no aplica hoy — solo guarda.

---

## Horarios de sueño

A diferencia de `strict_shedules` (una fila por hijo), `sleep_schedules` es una tabla **multi-fila** con hasta dos filas por hijo.

```
sleep_schedules
  bucket=weeknight  enabled=true   22:00 → 07:00  days=[1,2,3,4]
  bucket=weekend    enabled=true   23:00 → 08:30  days=[5,6,7]
```

Por qué dos filas:

- Lo habitual es tener hora de dormir distinta entre noches escolares y fines de semana.
- El array `days` es libre, así un padre que divide distinto (p. ej. L–S estricto, D flexible) puede reutilizar ambos huecos como quiera.
- Más de dos buckets sería raro. La restricción `unique(family_link_id, bucket)` impone el tope.

Upsert por `(family_link_id, bucket)` vía `PUT /parent/children/{id}/sleep-schedules`:

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

### Ventanas que cruzan medianoche

`end_time < start_time` significa que la ventana cruza la medianoche. El backend guarda ambas como `TIME` y la app cliente resuelve la fecha al aplicar — no hay lógica especial en BD.

### Días

Los días son enteros ISO-8601: `1 = Lunes`, `2 = Martes`, … `7 = Domingo`. Un array vacío deshabilita la fila (ningún día coincide). Mejor poner `enabled = false` en ese caso.

---

## Apps siempre disponibles

Columna JSON en `parental_controls`:

```json
"always_allowed_apps": [
  { "name": "Teléfono",  "emoji": "📞", "package": "com.android.phone" },
  { "name": "Mensajes",  "emoji": "💬" },
  { "name": "Maps",      "emoji": "🗺️" }
]
```

- `name` obligatorio, máx. 60 caracteres.
- `emoji` opcional, máx. 8 caracteres (cluster UTF-8).
- `package` opcional — id de paquete de Android o bundle id de iOS cuando la app hijo pueda resolver apps instaladas. Nullable hasta que esa capa esté construida.

Las apps de esta lista siguen disponibles durante las ventanas de sueño. También entrará en juego para el bloqueo por app (Fase B del plan), pero esa aplicación aún no está cableada.

Se escribe vía `PUT /parent/children/{id}/controls` con la clave `always_allowed_apps`. Omitirla deja la lista como está; enviar `null` la vacía.

---

## Strict schedules — la otra tabla de "schedule"

`strict_shedules` es anterior al trabajo parental y tiene otro propósito:

- **Una fila por hijo**, upserted sobre `user_id`.
- Guarda **siete toggles booleanos** para YouTube Shorts / Comentarios e Instagram Reels / Stories / Explore / etc.
- También tiene `start_time`, `end_time`, `days` — pero definen **cuándo aplican los toggles**, no un bloqueo global.

Cuando ambas tablas están configuradas para un hijo:

1. **Fuera** de cualquier ventana de sueño → aplica el filtro de `strict_shedules` en su propia `start_time`/`end_time`; el resto de apps sin restricción.
2. **Dentro** de una ventana de sueño → todo bloqueado salvo `always_allowed_apps`, independientemente de `strict_shedules`.

La app padre expone ambos como filas separadas en la pantalla de Reglas:

- **Tiempo de sueño** → escribe en `sleep_schedules`.
- **Redes sociales** → escribe en `strict_shedules` (endpoint existente `/parent/children/{id}/schedule`).

---

## Lo que **no** está aún en el esquema

- **Cola de comandos push** — no hay tabla que guarde comandos "lock now" / "unlock" pendientes. Necesita primero el servicio de envío FCM.
- **Cuotas por app** — los Límites por App (del plan) añadirán una tabla nueva (una fila por hijo+app+bucket). Aún sin construir.
- **Entitlement de suscripción** — no hay tabla `subscriptions`. La integración con RevenueCat añadirá una columna en `users` o una tabla dedicada.
- **Ubicación / geofencing** — no hay tablas de GPS. Add-on opcional del plan.

Ver [PLAN.md](../../../parent/PLAN.md) para la hoja de ruta.
