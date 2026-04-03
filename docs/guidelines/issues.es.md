# Formato de Issues y Pull Requests

Estos estándares se aplican a los tres repositorios (`app/`, `parent/`, `backend/`). Un formato consistente hace el triaje más rápido y mantiene el historial legible.

---

## Issues de GitHub

### Regla 1 — Formato del Título

```
tipo: frase corta
```

| Tipo          | Cuándo usarlo                                  |
| ------------- | ---------------------------------------------- |
| `feat`        | Nueva funcionalidad o pantalla                 |
| `bug`         | Algo está roto o se comporta incorrectamente   |
| `fix`         | Corrección puntual (alcance menor que un bug)  |
| `refactor`    | Reestructuración sin cambio de comportamiento  |
| `docs`        | Gaps o errores en documentación                |
| `enhancement` | Mejora de una funcionalidad existente          |

**Ejemplos:**

```
feat: Agregar widget de tiempo de pantalla diario en el inicio
bug: El OTP de login no llega a cuentas de Gmail
docs: Falta ejemplo de respuesta en la API de horarios
enhancement: Mejorar animación del gráfico de barras
```

---

### Regla 2 — Descripciones Bilingües

Cada descripción de issue debe estar escrita en **inglés y español**. Primero inglés, español inmediatamente después.

```markdown
## Description (EN)

We need to compress user avatars before uploading to S3 to reduce storage costs.

## Descripción (ES)

Necesitamos comprimir los avatares de los usuarios antes de subirlos a S3 para reducir costos de almacenamiento.
```

---

### Regla 3 — Especificaciones Técnicas

Para cualquier tarea que no sea una corrección trivial, incluir una sección de **Especificaciones Técnicas** con archivos concretos, endpoints, modelos y restricciones de lógica.

```markdown

- **File(s):** `lib/features/profile/data/rx_post_avatar/`, `ProfileUpdateController.php`
- **Endpoint / Model / Screen:** `POST /profile-avatar-upload`, `ProfileUpdateController`
- **Logic constraints:** Tamaño máximo 2 MB tras comprimir, solo salida JPEG
- **Dependencies or blockers:** Requiere subir `image_compress` a ^6.0
```

---

### Regla 4 — Tipo de Issue

Al abrir un issue en GitHub, selecciona un **Tipo**:

| Tipo        | Cuándo usarlo                                     |
| ----------- | ------------------------------------------------- |
| **Bug**     | Un problema inesperado o comportamiento roto      |
| **Feature** | Una solicitud, idea o nueva funcionalidad         |
| **Task**    | Una pieza de trabajo concreta — el tipo más común |

---

### Regla 5 — Requisitos de Etiquetas

Cada issue debe tener **una etiqueta de cada uno de los cinco grupos** antes de salir de Triaje.

#### Plataforma — ¿qué app afecta?

| Etiqueta     | Aplica a                              |
| ------------ | ------------------------------------- |
| `backend`    | API Laravel / panel de administración |
| `parent app` | Solo app Flutter para padres          |
| `user app`   | Solo app Flutter del niño (usuario)   |

#### SO — ¿qué sistema operativo?

| Etiqueta               | Aplica a                               |
| ---------------------- | -------------------------------------- |
| `android`              | Solo entorno Android                   |
| `iOS`                  | Solo entorno iOS                       |
| `both iOS and android` | Afecta a ambas plataformas móviles     |

#### Prioridad

| Etiqueta   | Significado                                          |
| ---------- | ---------------------------------------------------- |
| `critical` | El sitio está caído o una función principal está rota |
| `high`     | Importante para el próximo release                   |
| `medium`   | Tarea estándar                                       |
| `low`      | Bueno tener / backlog                                |

#### Sizing (Estimación de Esfuerzo)

| Etiqueta | Esfuerzo                  |
| -------- | ------------------------- |
| `S`      | 1–2 horas                 |
| `M`      | 1–3 días                  |
| `L`      | Una semana completa o más |
| `XL`     | Más de dos semanas        |

#### Tipo de Tarea

| Etiqueta           | Cuándo usarla                                        |
| ------------------ | ---------------------------------------------------- |
| `bug`              | Algo no funciona                                     |
| `feature`          | Nueva funcionalidad                                  |
| `enhancement`      | Solicitud de nueva función o mejora de algo existente |
| `refactor`         | Limpiar o reestructurar código                       |
| `documentation`    | Mejoras o adiciones a la documentación               |
| `good first issue` | Adecuado para nuevos colaboradores                   |

---

### Plantilla Completa del Issue

El archivo `.github/ISSUE_TEMPLATE/standard_task.md` en este repositorio pre-rellena todas las secciones automáticamente al abrir un nuevo issue en GitHub. El bloque de comentarios dentro de la plantilla contiene la referencia completa de etiquetas para consultarla mientras escribes.

---

## Pull Requests

### Formato del Título

Los títulos de PR siguen la misma convención de Conventional Commits usada en los mensajes de commit.

**App / Backend:**

```
tipo(alcance): descripción corta
```

| Tipo       | Cuándo usarlo                                  |
| ---------- | ---------------------------------------------- |
| `feat`     | Nueva funcionalidad                            |
| `fix`      | Corrección de bug                              |
| `chore`    | Dependencias, herramientas, configuración      |
| `refactor` | Reestructuración, sin cambio de comportamiento |
| `docs`     | Solo documentación                             |
| `test`     | Añadir o actualizar pruebas                    |
| `ci`       | Cambios en workflows de CI/CD                  |

**Ejemplos:**

```
feat(auth): add biometric login support
fix(controls): save button not updating lock
```

### Plantilla del Cuerpo del PR

```markdown
## Summary

<!-- What this PR does and why. -->

## Changes

-
-

## How to Test

<!-- Steps to verify the changes work correctly. -->

## Related Issue(s)

Closes #, Closes #

```

### Reglas

- Todos los checks de CI deben pasar antes de solicitar revisión
- Elimina la rama después de fusionar

---

## Nomenclatura de Ramas

### Desde un Issue de GitHub (preferido)

Usa el botón **"Create a branch"** en la página del issue. GitHub genera el nombre automáticamente:

```
<número-issue>-<título-issue-como-slug>
```

**Ejemplos:**
```
42-feat-add-daily-screen-time-widget
87-bug-login-otp-not-received-on-gmail
103-docs-missing-schedule-api-response-example
```

- Cambia siempre la rama base a `develop` en el diálogo antes de crear — GitHub usa `main` por defecto

### Sin un Issue Vinculado

Si el trabajo no tiene un issue asociado, usa la convención manual:

```
<iniciales>/<tipo>/<descripción-corta>
```

**Ejemplos:**
```
jc/feat/onboarding-screen
ab/fix/login-token-refresh
jc/docs/api-endpoint-reference
```

- Iniciales: 2–3 letras minúsculas, acordadas con el equipo
- Solo minúsculas y guiones
- Crear ramas desde `develop`, nunca desde `main`
