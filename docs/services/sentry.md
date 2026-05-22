# Error Reporting — Sentry

[Sentry](https://sentry.io/) captures unexpected exceptions across the three Back To Life codebases and surfaces them as grouped issues with stack traces, breadcrumbs, and (in time) release-tagged regressions.

## Three independent projects

Each codebase reports into its own Sentry project — easier alerting, cleaner platform settings (Laravel vs Flutter), no mixed quotas.

| Codebase | Sentry platform | DSN env var | What lands here |
|----------|-----------------|-------------|-----------------|
| Backend | Laravel | `SENTRY_LARAVEL_DSN` (in `.env`) | Unhandled PHP exceptions — DB errors, SMTP failures, null derefs in controllers, anything that escapes a `try` |
| User App | Flutter | `--dart-define=SENTRY_FLUTTER_DSN=<dsn>` (build flag) | Dart exceptions hitting `ToastUtil.showError` (catch blocks, future widget render errors) |
| Parent App | Flutter | `--dart-define=SENTRY_FLUTTER_DSN=<dsn>` (build flag) | Same as User App |

## What does **not** go to Sentry

By design, business outcomes never enter Sentry — they're not bugs and would burn the monthly quota.

**Backend ignores** (configured in [`config/sentry.php`](https://github.com/backtolife-app/backend/blob/main/config/sentry.php) `ignore_exceptions` + [`bootstrap/app.php`](https://github.com/backtolife-app/backend/blob/main/bootstrap/app.php) `dontReport`):

- `ValidationException` — 422
- `AuthenticationException` — 401
- `AuthorizationException` / `AccessDeniedHttpException` — 403
- `ThrottleRequestsException` — 429
- `ModelNotFoundException` / `NotFoundHttpException` — 404
- `MethodNotAllowedHttpException` — 405

**Flutter side** — `ToastUtil.showBackendError` does *not* call `Sentry.captureException`. The thinking:

- **5xx** is already in Sentry's backend project with full PHP context — no point capturing the Dio response too.
- **4xx** is a business outcome the user sees as a sanitized message — not a bug.

Only `ToastUtil.showError` (the generic "something unexpected threw" helper) reports to Sentry from Flutter.

## Quota strategy

Free tier is 5,000 errors/month per project. With current scale (~1,000 active users) and the 4xx filtering above, expected real-error volume is well under that. If you start seeing the quota wall, look at:

1. **Issue rate alerts in Sentry's UI.** Set `alert when X occurs > N times in 5 minutes` rather than rolling your own aggregation.
2. **What's slipping past the ignore list.** A new exception class that isn't 4xx-shaped but is still a business outcome should be added to `ignore_exceptions` + `dontReport`.

## 5xx response sanitization

Companion to Sentry reporting: the backend's exception handler ([`bootstrap/app.php`](https://github.com/backtolife-app/backend/blob/main/bootstrap/app.php)) returns a stable `{"message": "Something went wrong, please try again later.", "code": "internal_error"}` body for any 5xx that escapes a controller. The Flutter `showBackendError` then detects `status >= 500` and toasts a localized `commonGenericError` instead of trusting the body.

This kills the leak class where infrastructure errors (`535 5.7.8 authentication failed ...` from Brevo SMTP, `SQLSTATE[HY000]` from a DB hiccup, etc.) reached end users as raw toast text.

## Production turn-on

1. **Create three projects** at sentry.io: one Laravel project for the backend, two Flutter projects for the apps. Suggested names: `backend`, `user-app`, `parent-app`.
2. **Backend** — copy the Laravel DSN, add to deployed `.env`:
    ```
    SENTRY_LARAVEL_DSN=<your-dsn>
    ```
   Then `php artisan config:cache` so the cached config picks up the env value.
3. **User App + Parent App** — copy each Flutter DSN, add to the Codemagic build args (or any other CI you use for release builds):
    ```
    --dart-define=SENTRY_FLUTTER_DSN=<your-dsn>
    ```
   For local dev runs (`make waydroid-run` etc.), leave the DSN out of your `.env.json` — Sentry init becomes a no-op and you don't pollute production data with dev errors.

## Local dev behaviour

All three sides degrade cleanly when the DSN env var is unset:

- **Backend** — `sentry/sentry-laravel` is installed but never initialized; `report()` calls become no-ops.
- **Flutter** — `main.dart` skips the `SentryFlutter.init()` block entirely; `Sentry.captureException` calls become no-ops.

You can paste the DSN into a local `.env` (backend) or `.env.json` (Flutter) if you specifically want to test Sentry integration locally, but the default state is "off."
