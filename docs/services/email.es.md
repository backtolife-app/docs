# Proveedor de Email — Brevo

Todo el correo transaccional sale del backend a través de [Brevo](https://www.brevo.com/) (anteriormente Sendinblue). Se usa para:

- **Verificación OTP** al registrarse
- **OTP de restablecimiento de contraseña**
- **Email de reintento de registro** enviado por el comando artisan `users:cleanup-unverified`

## Plan y cuota

- **Plan actual:** 5.000 créditos/mes
- **Limitación de envío:** el comando artisan de limpieza envía a ~10/seg para no saturar el relay SMTP; los envíos transaccionales desde los endpoints API no están limitados (uno por petición)

Si se agota la cuota mensual de créditos, los envíos fallan silenciosamente desde la perspectiva del cliente — el bloque catch en [`RegisterController`](https://github.com/backtolife-app/backend/blob/main/app/Http/Controllers/API/Auth/RegisterController.php) ahora reporta el error SMTP a Sentry y devuelve un 500 genérico a la app. Vigila el saldo de créditos en el panel de Brevo y los issues "SMTP" en el proyecto backend de Sentry para saber cuándo te estás acercando al límite.

## Historial de migración

Hasta el **22-05-2026** el backend enviaba a través del relay SMTP incluido de Hostinger. Hostinger limita silenciosamente las cuentas de hosting compartido a ~500 correos/día; la tasa de envío de OTPs superaba ese límite, así que aproximadamente 1.059 usuarios registrados entre el 17-05-2026 y el 22-05-2026 nunca recibieron su código de verificación. Esas filas se eliminaron con el comando `users:cleanup-unverified` el día del cambio a Brevo.

Si estás investigando un reporte de "los usuarios no reciben el OTP", el saldo de créditos y la pestaña de rebotes/spam de Brevo son los primeros lugares donde mirar.

## Configuración

Claves `.env` del backend (todas obligatorias en prod, ninguna en control de versiones):

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

El entorno de desarrollo local usa por defecto `MAIL_MAILER=log` — cada envío se escribe en `storage/logs/laravel.log` como un cuerpo MIME completamente renderizado, no se envía ningún correo real.

## Renderizado por idioma

Desde el 22-05-2026 los correos de OTP y restablecimiento de contraseña se renderizan en **en / es / it** según un campo `locale` que los clientes Flutter envían en `/register`, `/forgot-password` y `/resend_otp`. Cuando el cliente omite el campo (versiones antiguas), el correo cae a un cuerpo bilingüe EN+ES.

Los Mailables son [`OTP`](https://github.com/backtolife-app/backend/blob/main/app/Mail/OTP.php) y [`SignupRetryAfterIssue`](https://github.com/backtolife-app/backend/blob/main/app/Mail/SignupRetryAfterIssue.php); las vistas Blade ramifican según `$locale` para el cuerpo y el método `envelope()` elige el asunto localizado mediante `match`.

## Lista de verificación de entregabilidad

- **SPF / DKIM / DMARC** deben estar configurados en el dominio remitente (`backtolife.site`) antes de salir a producción. El panel de Brevo guía a través de los registros DNS.
- **La dirección `From` debe coincidir con un dominio autenticado.** Enviar desde `no-reply@backtolife.site` solo funciona si `backtolife.site` está verificado en Brevo y firmado con DKIM.
- Vigila la pestaña de rebotes/quejas de Brevo. Una tasa de quejas por encima del 0,3 % en una ventana móvil pondrá la cuenta en modo de envío limitado.
