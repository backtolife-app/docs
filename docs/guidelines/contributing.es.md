# Guía de Contribución

Estos estándares se aplican a los tres repositorios (`app/`, `parent/`, `backend/`).

---

## Estrategia de Ramas

```
main       ← solo producción, protegida — nunca hacer push directamente
develop    ← rama de integración — todos los PRs se fusionan aquí
```

- Crea siempre ramas desde `develop`, nunca desde `main`
- Elimina las ramas después de fusionarlas
- Los PRs requieren al menos **1 revisión** antes de fusionarse

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

> Cambia siempre la rama base a `develop` en el diálogo antes de crear — GitHub usa `main` por defecto.

### Sin un Issue Vinculado

```
<iniciales>/<tipo>/<descripción-corta>
```

| Tipo | Cuándo usarlo |
|------|--------------|
| `feat/` | Nueva funcionalidad o pantalla |
| `fix/` | Corrección de errores |
| `chore/` | Dependencias, herramientas, configuración |
| `refactor/` | Reestructuración, sin cambio de comportamiento |
| `docs/` | Solo documentación |

**Ejemplos:**
```
jc/feat/onboarding-screen
ab/fix/login-token-refresh
jc/chore/update-flutter-sdk
jc/docs/api-endpoint-reference
```

Las iniciales deben ser 2–3 letras minúsculas, acordadas con el equipo, y mantenerse consistentes en todas las ramas y commits.

---

## Mensajes de Commit

Sigue [Conventional Commits](https://www.conventionalcommits.org/):

```
tipo(alcance): descripción corta
```

| Tipo | Cuándo usarlo |
|------|--------------|
| `feat` | Nueva funcionalidad |
| `fix` | Corrección de bug |
| `chore` | Build, dependencias, herramientas |
| `refactor` | Refactorización |
| `style` | Formato, sin cambio de lógica |
| `test` | Añadir o actualizar pruebas |
| `docs` | Documentación |
| `perf` | Mejora de rendimiento |
| `ci` | Cambios en CI/CD |

**Reglas:**
- Usa **modo imperativo** — "agregar", "corregir", "actualizar"
- Línea de asunto de menos de **72 caracteres**
- Sin punto al final
- Añade un cuerpo (línea en blanco tras el asunto) si el cambio necesita contexto

**Ejemplos:**
```
feat(auth): agregar soporte de login biométrico
fix(profile): corregir avatar que no carga en iOS
chore(deps): actualizar flutter a 3.22
refactor(home): extraer widget de gráfico a archivo separado
test(auth): añadir caso límite de expiración OTP
ci(backend): añadir paso de cobertura Pest
```

---

## Pull Requests

### Formato del Título

Mismo formato `tipo(alcance): descripción` que los mensajes de commit.

### Plantilla del Cuerpo

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

- Un tema por PR — divide los cambios no relacionados en PRs separados
- Todos los checks de CI deben pasar antes de solicitar revisión
- Resuelve todos los comentarios de revisión antes de fusionar
- Elimina la rama después de fusionar

---

## CI/CD

### App (`app/`)

Dos workflows de GitHub Actions — uno se ejecuta automáticamente, el otro se activa manualmente:

| Workflow | Activación | Pasos |
|----------|-----------|-------|
| App CI | Cada push / PR a `main` o `develop` | `flutter pub get` → `flutter analyze` → `flutter test` |
| iOS Simulator | Manual (solo `workflow_dispatch`) | `flutter pub get` → `flutter test` en runner macOS |

Las builds de release (App Store, Play Store) las gestiona **Codemagic** (`codemagic.yaml`). Pasa `APP_KEY_VALUE` como dart-define — nunca lo incluyas en el repositorio.

### App para Padres (`parent/`)

Un workflow de GitHub Actions se ejecuta en cada push o PR a `main` o `develop`:

| Paso | Comando |
|------|---------|
| Instalar dependencias | `flutter pub get` |
| Regenerar mocks | `dart run build_runner build --delete-conflicting-outputs` |
| Análisis estático | `flutter analyze --no-fatal-infos` |
| Pruebas | `flutter test --coverage` |

> El paso `build_runner` es obligatorio antes de `flutter test` — las pruebas importan archivos `*.mocks.dart` generados y fallarán al compilar sin ellos. Incluye los archivos `*.mocks.dart` en el commit junto con cualquier prueba que los use.

Las builds de release las gestiona **Codemagic** (`codemagic.yaml`).

### Backend (`backend/`)

Un workflow de GitHub Actions se ejecuta en cada push o PR a `main` o `develop`:

| Paso | Comando |
|------|---------|
| Instalar dependencias | `composer install` |
| Ejecutar pruebas | `./vendor/bin/pest` (SQLite en memoria) |

---

## Notas por Repositorio

### App (`app/`)
- Ejecuta `flutter analyze` localmente antes de hacer push — el CI impone cero advertencias
- Las pruebas están en `test/` divididas en `widget/`, `models/` y `helpers/`

### App para Padres (`parent/`)
- Después de añadir o cambiar una anotación `@GenerateMocks`, regenera los mocks antes de hacer commit:
  ```bash
  dart run build_runner build --delete-conflicting-outputs
  ```

### Backend (`backend/`)
- Las pruebas usan una base de datos SQLite en memoria — no se necesita Docker para ejecutarlas
- Ejecuta `php artisan test` o `./vendor/bin/pest` localmente antes de hacer push
