# BackToLife Docs

Bienvenido al centro de documentación centralizado de la plataforma **Back To Life**. Este sitio reúne la documentación de los tres repositorios en una única referencia con búsqueda integrada.

## Servicios

| Servicio | Descripción | Repositorio |
|----------|-------------|-------------|
| [App](app/index.md) | Aplicación móvil Flutter — cliente iOS y Android del niño | `app/` |
| [App para Padres](parent/index.md) | Aplicación Flutter para padres — panel, controles, gráficos de tiempo de pantalla | `parent/` |
| [Backend](backend/index.md) | API REST Laravel y panel de administración — compartido por ambas apps móviles | `backend/` |

## Pautas Globales

Estándares que se aplican en todos los repositorios y a todos los colaboradores.

| Guía | Descripción |
|------|-------------|
| [Formato de Issues y PRs](guidelines/issues.es.md) | Cómo redactar issues y pull requests en GitHub |
| [Panel del Proyecto](guidelines/project-board.es.md) | Cómo usar el panel de GitHub — columnas, flujo de tarjetas y automatización |
| [Contribución](guidelines/contributing.es.md) | Nomenclatura de ramas, commits, reglas de PRs y CI/CD por repositorio |

## Visión General de la Plataforma

```
App para Padres (Flutter)        App del Niño (Flutter)
        │                                │
        │  HTTPS + Fijación Certificados │
        └──────────┬─────────────────────┘
                   ▼
         API Backend (Laravel)
                   │
                   ▼
         Base de Datos MySQL 8.4
```

Ambas apps móviles comparten el mismo backend Laravel en `https://admin.backtolife.site/api`. Están registradas como apps separadas (`com.backtolife.parent` y `com.backtolife.app`) dentro del mismo proyecto Firebase.

## Stack en Resumen

| Capa | Tecnología |
|------|-----------|
| Móvil (niño) | Flutter — `com.backtolife.app` |
| Móvil (padre) | Flutter — `com.backtolife.parent` |
| API | Laravel 12, PHP 8.2+, autenticación Sanctum |
| Base de Datos | MySQL 8.4 |
| Notificaciones Push | Firebase Cloud Messaging (FCM / APNs) |
| CI/CD | GitHub Actions + Codemagic |
