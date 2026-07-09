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

### Regla 2 — Descríbelo desde el punto de vista del usuario

Escribe la descripción en lenguaje sencillo, desde la **perspectiva del usuario**: qué quiere hacer, qué debería ver o poder hacer, y por qué le importa. Describe el producto y la experiencia, no cómo construirlo. El "cómo" técnico (archivos, endpoints, código) lo resuelve quien tome la tarea, no aquí.

Una buena descripción responde:

- **Quién** la necesita — un padre, un niño, un administrador.
- **Qué** quiere hacer, o el problema que tiene hoy.
- **Qué debería pasar** — el comportamiento o resultado esperado.
- **Por qué** le importa.

Escríbela en **inglés y español** — primero inglés, español inmediatamente después.

```markdown
## Description (EN)

As a parent, I want to see how much screen time my child used today right on the home screen, so I can check it at a glance without opening any menus. Today I have to go into the stats page every time, which is slow and easy to forget.

## Descripción (ES)

Como padre, quiero ver cuánto tiempo de pantalla usó mi hijo hoy directamente en la pantalla de inicio, para poder revisarlo de un vistazo sin abrir ningún menú. Hoy tengo que entrar a la página de estadísticas cada vez, lo cual es lento y fácil de olvidar.
```

---

### Regla 3 — Tipo de Issue

Al abrir un issue en GitHub, selecciona un **Tipo**:

| Tipo        | Cuándo usarlo                                     |
| ----------- | ------------------------------------------------- |
| **Bug**     | Un problema inesperado o comportamiento roto      |
| **Feature** | Una solicitud, idea o nueva funcionalidad         |
| **Task**    | Una pieza de trabajo concreta — el tipo más común |

---

### Regla 4 — Requisitos de Etiquetas

Las etiquetas se seleccionan en la **barra lateral de GitHub** (lado derecho del issue), no se escriben en el cuerpo. Cada issue debe tener **una etiqueta de cada uno de los cinco grupos** — incluido el **Tamaño** (`S`/`M`/`L`/`XL`) — antes de salir de Triaje.

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

Cuánto se espera que tome la tarea. **Estima siempre con la columna Tiempo completo.** La columna Medio tiempo es el tiempo de calendario equivalente para alguien que trabaja ~2 días por semana, mostrada solo como referencia de planificación — nunca estimes directamente con ella.

| Etiqueta | Tiempo completo (referencia) | Medio tiempo (~2 días/semana) |
| -------- | ---------------------------- | ----------------------------- |
| `S`      | 1 día de calendario          | 3 días de calendario          |
| `M`      | 3 días de calendario         | 7 días de calendario          |
| `L`      | 7 días de calendario         | 14 días de calendario         |
| `XL`     | 14 días de calendario        | 28 días de calendario         |

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

### Ejemplo Completo

Un issue completo. Solo el **título** y la **descripción** van en el cuerpo — todo lo demás se selecciona en la **barra lateral** de GitHub.

**Título** (cuerpo)

```
feat: Show today's screen time on the parent home screen
```

**Descripción** (cuerpo)

```markdown
## Description (EN)

As a parent, I want to see how much screen time my child used today right on the home screen, so I can check it at a glance without opening any menus. Today I have to go into the stats page every time, which is slow and easy to forget.

## Descripción (ES)

Como padre, quiero ver cuánto tiempo de pantalla usó mi hijo hoy directamente en la pantalla de inicio, para poder revisarlo de un vistazo sin abrir ningún menú. Hoy tengo que entrar a la página de estadísticas cada vez, lo cual es lento y fácil de olvidar.
```

**Barra lateral** (no el cuerpo)

- **Type:** Feature — **Projects:** BackToLife
- **Etiquetas:** `parent app` · `both iOS and android` · `medium` · `M` · `feature`

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
