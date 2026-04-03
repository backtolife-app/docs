# BackToLife Docs

Welcome to the centralised documentation hub for the **Back To Life** platform. This site pulls together documentation from all three repositories into a single, searchable reference.

## Services

| Service | Description | Repository |
|---------|-------------|------------|
| [App](app/index.md) | Flutter mobile app — iOS & Android child-side client | `app/` |
| [Parent App](parent/index.md) | Flutter parent companion app — dashboard, controls, screen time charts | `parent/` |
| [Backend](backend/index.md) | Laravel REST API & admin panel — shared by both mobile apps | `backend/` |

## Global Guidelines

Standards that apply across all repositories and all contributors.

| Guide | Description |
|-------|-------------|
| [Issue & PR Format](guidelines/issues.md) | How to write GitHub issues and pull requests |
| [Project Board](guidelines/project-board.md) | How to use the GitHub project board — columns, card flow, automation |
| [Contributing](guidelines/contributing.md) | Branch naming, commit messages, PR rules, and CI/CD per repo |

## Platform Overview

```
Parent App (Flutter)          Child App (Flutter)
        │                             │
        │  HTTPS + Certificate Pin    │
        └──────────┬──────────────────┘
                   ▼
         Backend API (Laravel)
                   │
                   ▼
           MySQL 8.4 Database
```

Both mobile apps share the same Laravel backend at `https://admin.backtolife.site/api`. They are registered as separate apps (`com.backtolife.parent` and `com.backtolife.app`) within the same Firebase project.

## Stack at a Glance

| Layer | Technology |
|-------|-----------|
| Mobile (child) | Flutter — `com.backtolife.app` |
| Mobile (parent) | Flutter — `com.backtolife.parent` |
| API | Laravel 12, PHP 8.2+, Sanctum auth |
| Database | MySQL 8.4 |
| Push Notifications | Firebase Cloud Messaging (FCM / APNs) |
| CI/CD | GitHub Actions + Codemagic |
