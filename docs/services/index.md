# Services

External SaaS providers the Back To Life platform depends on. Each service has its own page covering what it's used for, where it's configured, and how to turn it on for new environments.

## Current services

| Service | Provider | Used by | Purpose |
|---------|----------|---------|---------|
| [Email Provider](email.md) | Brevo | Backend | Transactional email — OTP, password reset, signup-retry apology |
| [Error Reporting](sentry.md) | Sentry | Backend, User App, Parent App | Unexpected-exception capture and alerting |

## Conventions

- Credentials and DSNs **never** live in version control. Each service exposes its secret through one or more environment variables that are set per-environment (`.env` for the backend, `--dart-define` for the Flutter apps).
- When unset, every service degrades cleanly to "off" in local dev — Sentry init is a no-op, mail falls back to the `log` driver, etc. Production turn-on is a single env-var change.
- New service entries belong here too. Add a subpage, then list it in the table above.
