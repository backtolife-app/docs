# Email Provider — Brevo

All transactional email leaves the backend through [Brevo](https://www.brevo.com/) (formerly Sendinblue). Used for:

- **OTP verification** on signup
- **Password reset** OTP
- **Signup-retry apology** sent by the `users:cleanup-unverified` artisan command

## Plan & quota

- **Current plan:** 5,000 credits/month
- **Sending throttle:** the cleanup artisan command paces sends at ~10/sec to stay friendly with the SMTP relay; transactional sends from the API endpoints are unthrottled (one-at-a-time per request)

If the monthly credit cap is hit, sends fail silently from the client's perspective — the catch block in [`RegisterController`](https://github.com/backtolife-app/backend/blob/main/app/Http/Controllers/API/Auth/RegisterController.php) now reports the SMTP error to Sentry and returns a generic 500 to the app. Watch the Brevo dashboard's credit balance + the Sentry backend project's "SMTP" issues to know when you're getting close.

## Migration history

Until **2026-05-22** the backend sent through Hostinger's bundled SMTP relay. Hostinger silently caps shared-hosting accounts at ~500 emails/day; the OTP send rate exceeded that, so roughly 1,059 users registered between 2026-05-17 and 2026-05-22 never received their verification code. Those rows were swept by the `users:cleanup-unverified` command on the day of the Brevo switch.

If you're investigating a "users not getting OTP" report, the credit balance + Brevo's bounce/spam tab are the first places to look.

## Configuration

Backend `.env` keys (all required in prod, none in version control):

```
MAIL_MAILER=smtp
MAIL_HOST=smtp-relay.brevo.com
MAIL_PORT=587
MAIL_USERNAME=<brevo-smtp-login>
MAIL_PASSWORD=<brevo-smtp-key>
MAIL_ENCRYPTION=tls
MAIL_FROM_ADDRESS=no-reply@backtolife.site
MAIL_FROM_NAME="BackToLife"
```

Local dev defaults to `MAIL_MAILER=log` — every send is written to `storage/logs/laravel.log` as a fully-rendered MIME body, no real email is dispatched.

## Per-locale rendering

Since 2026-05-22 the OTP and password-reset emails render in **en / es / it** based on a `locale` field the Flutter clients pass on `/register`, `/forgot-password`, and `/resend_otp`. When the client omits the field (older builds) the email falls back to a bilingual EN+ES body.

The Mailables are [`OTP`](https://github.com/backtolife-app/backend/blob/main/app/Mail/OTP.php) and [`SignupRetryAfterIssue`](https://github.com/backtolife-app/backend/blob/main/app/Mail/SignupRetryAfterIssue.php); the Blade views branch on `$locale` for the body and the `envelope()` method picks the localized subject via `match`.

## Deliverability checklist

- **SPF / DKIM / DMARC** must be configured on the sender domain (`backtolife.site`) before going live. Brevo's dashboard walks through the DNS records.
- **`From` address must match an authenticated domain.** Sending from `no-reply@backtolife.site` only works if `backtolife.site` is verified in Brevo and DKIM-signed.
- Watch the Brevo bounce/complaint tab. A complaint rate above 0.3 % over a rolling window will land the account in throttled-send mode.
