# Email Provider — MailerSend

All transactional email leaves the backend through [MailerSend](https://www.mailersend.com/). Used for:

- **OTP verification** on signup
- **Password reset** OTP

A secondary `support` mailer (Hostinger SMTP) handles human support correspondence. Mailables without an explicit `->mailer()` call use the default mailer (MailerSend).

## Plan & quota

- **Free tier:** 3,000 emails/month, then a paid plan bills at roughly $1.20 per 1,000 emails.
- **Daily request quota** resets at midnight UTC.
- **Bulk endpoint** (`/v1/bulk-email`) accepts up to 500 recipients per request, for any future batch / marketing sends.

If the cap is hit, sends fail from the client's perspective. The catch block in [`RegisterController`](https://github.com/backtolife-app/backend/blob/main/app/Http/Controllers/API/Auth/RegisterController.php) reports the SMTP error to Sentry and returns a generic 500 to the app. Watch the MailerSend dashboard's activity / bounce tabs and the Sentry backend project's "SMTP" issues to know when you are getting close.

## Migration history

- Until **2026-05-22** the backend sent through Hostinger's bundled SMTP relay. Hostinger silently caps shared-hosting accounts at ~500 emails/day; the OTP send rate exceeded that, so a batch of users registered between 2026-05-17 and 2026-05-22 never received their verification code.
- From **2026-05-22** the backend used Brevo. Brevo's SMTP authentication began failing in production (`535 5.7.8 Authentication failed`), which silently broke OTP delivery, so the backend was switched to MailerSend.
- Since the MailerSend switch, transactional mail runs through `smtp.mailersend.net`.

If you are investigating a "users not getting OTP" report, the MailerSend activity log + bounce/spam tab are the first places to look. Sender reputation is shared with OTP deliverability, so keep the bounce rate low (validate addresses before any bulk send).

## Configuration

Backend `.env` keys (all required in prod, none in version control):

```
MAIL_MAILER=smtp
MAIL_HOST=smtp.mailersend.net
MAIL_PORT=587
MAIL_USERNAME=<mailersend-smtp-login>
MAIL_PASSWORD=<mailersend-smtp-password>
MAIL_ENCRYPTION=tls
MAIL_FROM_ADDRESS=no-reply@backtolife.site
MAIL_FROM_NAME="BackToLife"

# API token for the bulk-email endpoint (services.mailersend.token).
# Used by a planned admin marketing-email interface; transactional SMTP
# does not depend on it.
MAILERSEND_API_TOKEN=<mailersend-api-token>
```

Local dev defaults to `MAIL_MAILER=log` so every send is written to `storage/logs/laravel.log` as a fully-rendered MIME body, with no real email dispatched.

## Per-locale rendering

The OTP and password-reset emails render in **en / es / it** based on a `locale` field the Flutter clients pass on `/register`, `/forgot-password`, and `/resend_otp`. When the client omits the field (older builds) the email falls back to a bilingual EN+ES body.

The Mailable is [`OTP`](https://github.com/backtolife-app/backend/blob/main/app/Mail/OTP.php); the Blade view branches on `$locale` for the body and the `envelope()` method picks the localized subject via `match`.

## Deliverability checklist

- **SPF / DKIM / DMARC** must be configured on the sender domain (`backtolife.site`). The DNS should include `include:_spf.mailersend.net` in the SPF record, the MailerSend DKIM records, and a DMARC record. MailerSend's dashboard walks through the exact records.
- **`From` address must match an authenticated domain.** Sending from `no-reply@backtolife.site` only works if `backtolife.site` is verified in MailerSend and DKIM-signed.
- Watch the MailerSend bounce/complaint tab. A high bounce or complaint rate degrades sender reputation, which directly hurts OTP deliverability.
