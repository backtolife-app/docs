# Proveedor de Email — MailerSend

Todo el correo transaccional sale del backend a través de [MailerSend](https://www.mailersend.com/). Se usa para:

- **Verificación OTP** al registrarse
- **OTP de restablecimiento de contraseña**

Un mailer secundario `support` (SMTP de Hostinger) gestiona la correspondencia de soporte. Los Mailables sin una llamada explícita a `->mailer()` usan el mailer por defecto (MailerSend).

## Plan y cuota

- **Nivel gratuito:** 3.000 correos/mes; luego un plan de pago factura alrededor de $1,20 por cada 1.000 correos.
- **La cuota diaria de peticiones** se reinicia a medianoche UTC.
- **El endpoint de envío masivo** (`/v1/bulk-email`) acepta hasta 500 destinatarios por petición, para futuros envíos por lotes / marketing.

Si se agota la cuota, los envíos fallan desde la perspectiva del cliente. El bloque catch en [`RegisterController`](https://github.com/backtolife-app/backend/blob/main/app/Http/Controllers/API/Auth/RegisterController.php) reporta el error SMTP a Sentry y devuelve un 500 genérico a la app. Vigila las pestañas de actividad / rebotes del panel de MailerSend y los issues "SMTP" en el proyecto backend de Sentry para saber cuándo te estás acercando al límite.

## Historial de migración

- Hasta el **22-05-2026** el backend enviaba a través del relay SMTP incluido de Hostinger. Hostinger limita silenciosamente las cuentas de hosting compartido a ~500 correos/día; la tasa de envío de OTPs superaba ese límite, así que un grupo de usuarios registrados entre el 17-05-2026 y el 22-05-2026 nunca recibió su código de verificación.
- Desde el **22-05-2026** el backend usó Brevo. La autenticación SMTP de Brevo empezó a fallar en producción (`535 5.7.8 Authentication failed`), lo que rompió silenciosamente la entrega de OTPs, así que el backend se cambió a MailerSend.
- Desde el cambio a MailerSend, el correo transaccional pasa por `smtp.mailersend.net`.

Si estás investigando un reporte de "los usuarios no reciben el OTP", el registro de actividad de MailerSend y la pestaña de rebotes/spam son los primeros lugares donde mirar. La reputación del remitente es compartida con la entregabilidad de los OTP, así que mantén baja la tasa de rebotes (valida las direcciones antes de cualquier envío masivo).

## Configuración

Claves `.env` del backend (todas obligatorias en prod, ninguna en control de versiones):

```
MAIL_MAILER=smtp
MAIL_HOST=smtp.mailersend.net
MAIL_PORT=587
MAIL_USERNAME=<mailersend-smtp-login>
MAIL_PASSWORD=<mailersend-smtp-password>
MAIL_ENCRYPTION=tls
MAIL_FROM_ADDRESS=no-reply@backtolife.site
MAIL_FROM_NAME="BackToLife"

# Token de API para el endpoint de envío masivo (services.mailersend.token).
# Usado por una futura interfaz de correo de marketing en el admin; el SMTP
# transaccional no depende de él.
MAILERSEND_API_TOKEN=<mailersend-api-token>
```

El entorno de desarrollo local usa por defecto `MAIL_MAILER=log`, así cada envío se escribe en `storage/logs/laravel.log` como un cuerpo MIME completamente renderizado, sin enviar ningún correo real.

## Renderizado por idioma

Los correos de OTP y restablecimiento de contraseña se renderizan en **en / es / it** según un campo `locale` que los clientes Flutter envían en `/register`, `/forgot-password` y `/resend_otp`. Cuando el cliente omite el campo (versiones antiguas), el correo cae a un cuerpo bilingüe EN+ES.

El Mailable es [`OTP`](https://github.com/backtolife-app/backend/blob/main/app/Mail/OTP.php); la vista Blade ramifica según `$locale` para el cuerpo y el método `envelope()` elige el asunto localizado mediante `match`.

## Lista de verificación de entregabilidad

- **SPF / DKIM / DMARC** deben estar configurados en el dominio remitente (`backtolife.site`). El DNS debe incluir `include:_spf.mailersend.net` en el registro SPF, los registros DKIM de MailerSend y un registro DMARC. El panel de MailerSend guía a través de los registros exactos.
- **La dirección `From` debe coincidir con un dominio autenticado.** Enviar desde `no-reply@backtolife.site` solo funciona si `backtolife.site` está verificado en MailerSend y firmado con DKIM.
- Vigila la pestaña de rebotes/quejas de MailerSend. Una tasa alta de rebotes o quejas degrada la reputación del remitente, lo que perjudica directamente la entregabilidad de los OTP.
