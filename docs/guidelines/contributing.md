# Contributing Guide

These standards apply to all three repositories (`app/`, `parent/`, `backend/`).

---

## Branch Strategy

```
main       ← production only, protected — never push directly
develop    ← integration branch — all PRs merge here
```

- Always branch off `develop`, never off `main`
- Delete branches after merging
- PRs require at least **1 review** before merging

---

## Branch Naming

### From a GitHub Issue (preferred)

Use the **"Create a branch"** button on the issue page. GitHub generates the name automatically:

```
<issue-number>-<issue-title-as-slug>
```

**Examples:**
```
42-feat-add-daily-screen-time-widget
87-bug-login-otp-not-received-on-gmail
103-docs-missing-schedule-api-response-example
```

> Always change the base branch to `develop` in the dialog before creating — GitHub defaults to `main`.

### Without a Linked Issue

```
<initials>/<type>/<short-description>
```

| Type | When to use |
|------|-------------|
| `feat/` | New feature or screen |
| `fix/` | Bug fix |
| `chore/` | Dependencies, tooling, config |
| `refactor/` | Code restructure, no behaviour change |
| `docs/` | Documentation only |

**Examples:**
```
jc/feat/onboarding-screen
ab/fix/login-token-refresh
jc/chore/update-flutter-sdk
jc/docs/api-endpoint-reference
```

Initials must be 2–3 lowercase letters, agreed with the team, and kept consistent across all branches and commits.

---

## Commit Messages

Follow [Conventional Commits](https://www.conventionalcommits.org/):

```
type(scope): short description
```

| Type | When to use |
|------|-------------|
| `feat` | New feature |
| `fix` | Bug fix |
| `chore` | Build, deps, tooling |
| `refactor` | Refactoring |
| `style` | Formatting, no logic change |
| `test` | Adding or updating tests |
| `docs` | Documentation |
| `perf` | Performance improvement |
| `ci` | CI/CD changes |

**Rules:**
- Use **imperative mood** — "add", "fix", "update" (not "added", "fixing")
- Keep subject line under **72 characters**
- No period at the end
- Add a body (blank line after subject) if the change needs more context

**Examples:**
```
feat(auth): add biometric login support
fix(profile): correct avatar not loading on iOS
chore(deps): upgrade flutter to 3.22
refactor(home): extract chart widget into separate file
test(auth): add OTP expiry edge case
ci(backend): add Pest coverage report step
```

---

## Pull Requests

### Title Format

Same `type(scope): description` format as commit messages.

### Body Template

```markdown
## Summary

<!-- What this PR does and why. -->

## Changes

-
-

## How to Test

<!-- Steps to verify the changes work correctly. -->

## Related Issue(s)

Closes #, Closes # (after the # needs to be the number of the issue)
```

### Rules

- All CI checks must pass before requesting review
- Delete the branch after merging

---

## CI/CD

### App (`app/`)

Two GitHub Actions workflows — one runs automatically, one is triggered manually:

| Workflow | Trigger | Steps |
|----------|---------|-------|
| App CI | Every push / PR to `main` or `develop` | `flutter pub get` → `flutter analyze` → `flutter test` |
| iOS Simulator | Manual (`workflow_dispatch` only) | `flutter pub get` → `flutter test` on macOS runner |

Release builds (App Store, Play Store) are handled by **Codemagic** (`codemagic.yaml`). Pass `APP_KEY_VALUE` as a dart-define — never commit it.

### Parent App (`parent/`)

One GitHub Actions workflow runs on every push or PR to `main` or `develop`:

| Step | Command |
|------|---------|
| Install deps | `flutter pub get` |
| Regenerate mocks | `dart run build_runner build --delete-conflicting-outputs` |
| Static analysis | `flutter analyze --no-fatal-infos` |
| Tests | `flutter test --coverage` |

> The `build_runner` step is required before `flutter test` — tests import generated `*.mocks.dart` files and will fail to compile without it. Commit the `*.mocks.dart` files alongside any test that uses them.

Release builds are handled by **Codemagic** (`codemagic.yaml`).

### Backend (`backend/`)

One GitHub Actions workflow runs on every push or PR to `main` or `develop`:

| Step | Command |
|------|---------|
| Install deps | `composer install` |
| Run tests | `./vendor/bin/pest` (SQLite in-memory) |

---

## Repo-Specific Notes

### App (`app/`)
- Run `flutter analyze` locally before pushing — the CI enforces zero warnings
- Tests live in `test/` split into `widget/`, `models/`, and `helpers/`

### Parent App (`parent/`)
- After adding or changing a `@GenerateMocks` annotation, regenerate mocks before committing:
  ```bash
  dart run build_runner build --delete-conflicting-outputs
  ```

### Backend (`backend/`)
- Tests use an SQLite in-memory database — no Docker needed to run them
- Run `php artisan test` or `./vendor/bin/pest` locally before pushing
