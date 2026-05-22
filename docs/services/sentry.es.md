# Reporte de Errores — Sentry

[Sentry](https://sentry.io/) captura excepciones inesperadas en los tres repositorios de Back To Life y las muestra como issues agrupados con stack traces, breadcrumbs y (con el tiempo) regresiones etiquetadas por release.

## Tres proyectos independientes

Cada repositorio reporta a su propio proyecto Sentry — más fácil para las alertas, configuración de plataforma más limpia (Laravel vs Flutter), cuotas no mezcladas.

| Repositorio | Plataforma Sentry | Variable de entorno DSN | Qué llega aquí |
|-------------|-------------------|-------------------------|----------------|
| Backend | Laravel | `SENTRY_LARAVEL_DSN` (en `.env`) | Excepciones PHP no manejadas — errores de BD, fallos SMTP, null derefs en controladores, cualquier cosa que escape de un `try` |
| App de Usuario | Flutter | `--dart-define=SENTRY_FLUTTER_DSN=<dsn>` (flag de build) | Excepciones Dart que llegan a `ToastUtil.showError` (bloques catch, futuros errores de render de widgets) |
| App de Padres | Flutter | `--dart-define=SENTRY_FLUTTER_DSN=<dsn>` (flag de build) | Igual que la App de Usuario |

## Qué **no** va a Sentry

Por diseño, los resultados de negocio nunca entran en Sentry — no son bugs y quemarían la cuota mensual.

**El backend ignora** (configurado en [`config/sentry.php`](https://github.com/backtolife-app/backend/blob/main/config/sentry.php) `ignore_exceptions` + [`bootstrap/app.php`](https://github.com/backtolife-app/backend/blob/main/bootstrap/app.php) `dontReport`):

- `ValidationException` — 422
- `AuthenticationException` — 401
- `AuthorizationException` / `AccessDeniedHttpException` — 403
- `ThrottleRequestsException` — 429
- `ModelNotFoundException` / `NotFoundHttpException` — 404
- `MethodNotAllowedHttpException` — 405

**Lado Flutter** — `ToastUtil.showBackendError` *no* llama a `Sentry.captureException`. El razonamiento:

- **5xx** ya está en el proyecto backend de Sentry con contexto PHP completo — no tiene sentido capturar también la respuesta Dio.
- **4xx** es un resultado de negocio que el usuario ve como un mensaje sanitizado — no es un bug.

Solo `ToastUtil.showError` (el helper genérico de "algo inesperado lanzó una excepción") reporta a Sentry desde Flutter.

## Estrategia de cuota

El plan gratuito son 5.000 errores/mes por proyecto. Con la escala actual (~1.000 usuarios activos) y el filtrado de 4xx anterior, el volumen real de errores esperado está muy por debajo de eso. Si empiezas a chocar contra el límite de cuota, mira:

1. **Alertas de tasa de issues en la UI de Sentry.** Configura `alerta cuando X ocurra > N veces en 5 minutos` en lugar de hacer tu propia agregación.
2. **Qué se está escapando de la lista de ignorados.** Una nueva clase de excepción que no tiene forma de 4xx pero sigue siendo un resultado de negocio debe añadirse a `ignore_exceptions` + `dontReport`.

## Saneamiento de respuestas 5xx

Complemento del reporte a Sentry: el manejador de excepciones del backend ([`bootstrap/app.php`](https://github.com/backtolife-app/backend/blob/main/bootstrap/app.php)) devuelve un cuerpo estable `{"message": "Something went wrong, please try again later.", "code": "internal_error"}` para cualquier 5xx que escape de un controlador. Después, el `showBackendError` de Flutter detecta `status >= 500` y muestra un `commonGenericError` localizado en lugar de confiar en el cuerpo de la respuesta.

Esto cierra la clase de fugas donde errores de infraestructura (`535 5.7.8 authentication failed ...` del SMTP de Brevo, `SQLSTATE[HY000]` de un hipo de la BD, etc.) llegaban a los usuarios como texto en un toast.

## Activación en producción

1. **Crea tres proyectos** en sentry.io: un proyecto Laravel para el backend, dos proyectos Flutter para las apps. Nombres sugeridos: `backend`, `user-app`, `parent-app`.
2. **Backend** — copia el DSN de Laravel, añádelo al `.env` desplegado:
    ```
    SENTRY_LARAVEL_DSN=<tu-dsn>
    ```
   Después `php artisan config:cache` para que la configuración cacheada coja el valor del entorno.
3. **App de Usuario + App de Padres** — copia cada DSN de Flutter, añádelo a los argumentos de build de Codemagic (o cualquier otra CI que uses para builds de release):
    ```
    --dart-define=SENTRY_FLUTTER_DSN=<tu-dsn>
    ```
   Para ejecuciones de desarrollo local (`make waydroid-run` etc.), deja el DSN fuera de tu `.env.json` — la inicialización de Sentry se convierte en no-op y no contaminas los datos de producción con errores de desarrollo.

## Comportamiento en desarrollo local

Los tres lados se degradan limpiamente cuando la variable de entorno del DSN no está configurada:

- **Backend** — `sentry/sentry-laravel` está instalado pero nunca se inicializa; las llamadas a `report()` se convierten en no-ops.
- **Flutter** — `main.dart` salta el bloque `SentryFlutter.init()` entero; las llamadas a `Sentry.captureException` se convierten en no-ops.

Puedes pegar el DSN en un `.env` local (backend) o `.env.json` (Flutter) si específicamente quieres probar la integración de Sentry localmente, pero el estado por defecto es "apagado".
