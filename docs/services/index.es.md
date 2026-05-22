# Servicios

Proveedores SaaS externos de los que depende la plataforma Back To Life. Cada servicio tiene su propia página que cubre para qué se usa, dónde se configura y cómo activarlo en nuevos entornos.

## Servicios actuales

| Servicio | Proveedor | Usado por | Propósito |
|----------|-----------|-----------|-----------|
| [Proveedor de Email](email.md) | Brevo | Backend | Email transaccional — OTP, restablecimiento de contraseña, email de reintento de registro |
| [Reporte de Errores](sentry.md) | Sentry | Backend, App de Usuario, App de Padres | Captura y alertas de excepciones inesperadas |

## Convenciones

- Las credenciales y DSN **nunca** se guardan en control de versiones. Cada servicio expone su secreto mediante una o más variables de entorno configuradas por entorno (`.env` para el backend, `--dart-define` para las apps Flutter).
- Cuando no están configurados, todos los servicios se degradan limpiamente a "apagado" en desarrollo local — la inicialización de Sentry es un no-op, el correo cae al driver `log`, etc. La activación en producción es un único cambio de variable de entorno.
- Las nuevas entradas de servicios también van aquí. Añade una subpágina y luego inclúyela en la tabla de arriba.
