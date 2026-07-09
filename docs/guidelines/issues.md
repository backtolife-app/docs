# Issue & Pull Request Format

These standards apply to all three repositories (`app/`, `parent/`, `backend/`). Consistent formatting makes triage faster and keeps the history readable.

---

## GitHub Issues

### Rule 1 — Title Format

```
type: one short phrase
```

| Type          | When to use                                    |
| ------------- | ---------------------------------------------- |
| `feat`        | New feature or screen                          |
| `bug`         | Something is broken or behaving incorrectly    |
| `fix`         | Targeted correction (smaller scope than a bug) |
| `refactor`    | Code restructure with no behaviour change      |
| `docs`        | Documentation gaps or errors                   |
| `enhancement` | Improvement to an existing feature             |

**Examples:**

```
feat: Add daily screen time widget to home screen
bug: Login OTP not received on Gmail addresses
docs: Missing schedule API response example
enhancement: Improve chart animation performance
```

---

### Rule 2 — Describe it from the user's point of view

Write the description in plain language, from the **user's perspective** — what they want to do, what they should see or be able to do, and why it matters to them. Describe the product and the experience, not how to build it. The technical "how" (files, endpoints, code) is worked out by whoever picks up the task, not here.

A good description answers:

- **Who** it's for — a parent, a child, an admin.
- **What** they want to do, or the problem they run into today.
- **What should happen** — the expected behaviour or result.
- **Why** it matters to them.

Write it in **both English and Spanish** — English first, Spanish immediately after.

```markdown
## Description (EN)

As a parent, I want to see how much screen time my child used today right on the home screen, so I can check it at a glance without opening any menus. Today I have to go into the stats page every time, which is slow and easy to forget.

## Descripción (ES)

Como padre, quiero ver cuánto tiempo de pantalla usó mi hijo hoy directamente en la pantalla de inicio, para poder revisarlo de un vistazo sin abrir ningún menú. Hoy tengo que entrar a la página de estadísticas cada vez, lo cual es lento y fácil de olvidar.
```

---

### Rule 3 — Issue Type

When opening an issue on GitHub, select one **Type**:

| Type        | When to use                                 |
| ----------- | ------------------------------------------- |
| **Bug**     | An unexpected problem or broken behaviour   |
| **Feature** | A request, idea, or new functionality       |
| **Task**    | A specific piece of work — most common type |

---

### Rule 4 — Label Requirements

Every issue must have **one label from each of the five groups** before being moved out of Triage.

#### Platform — which app does this affect?

| Label        | Applies to                    |
| ------------ | ----------------------------- |
| `backend`    | Laravel API / admin panel     |
| `parent app` | Flutter parent app only       |
| `user app`   | Flutter child (user) app only |

#### OS — which operating system?

| Label                  | Applies to                    |
| ---------------------- | ----------------------------- |
| `android`              | Android environment only      |
| `iOS`                  | iOS environment only          |
| `both iOS and android` | Affects both mobile platforms |

#### Priority

| Label      | Meaning                                  |
| ---------- | ---------------------------------------- |
| `critical` | Site is down or a core feature is broken |
| `high`     | Important for the next release           |
| `medium`   | Standard task                            |
| `low`      | Nice-to-have / backlog                   |

#### Sizing (Effort Estimate)

How long the task is expected to take. **Always size against the Full-time column.** The Part-time column is the equivalent wall-clock time for someone working ~2 days a week and is shown for scheduling reference only — never estimate directly from it.

| Label | Full-time (reference) | Part-time (~2 days/week) |
| ----- | --------------------- | ------------------------ |
| `S`   | 1 calendar day        | 3 calendar days          |
| `M`   | 3 calendar days       | 7 calendar days          |
| `L`   | 7 calendar days       | 14 calendar days         |
| `XL`  | 14 calendar days      | 28 calendar days         |

#### Task Type

| Label              | When to use                                              |
| ------------------ | -------------------------------------------------------- |
| `bug`              | Something isn't working                                  |
| `feature`          | New functionality                                        |
| `enhancement`      | New feature request or improvement to existing behaviour |
| `refactor`         | Cleaning up or restructuring code                        |
| `documentation`    | Improvements or additions to documentation               |
| `good first issue` | Good for newcomers                                       |

---

### Full Issue Template

The `.github/ISSUE_TEMPLATE/standard_task.md` file in this repository pre-fills all sections automatically when you open a new issue on GitHub. The comment block inside the template contains the full label reference so it is always available while writing.

---

## Pull Requests

### Title Format

PR titles follow the same Conventional Commits convention used in commit messages.

**App / Backend:**

```
type(scope): short description
```

| Type        | When to use                           |
| ----------- | ------------------------------------- |
| `feat`      | New feature                           |
| `fix`       | Bug fix                               |
| `chore`     | Deps, tooling, config                 |
| `refactor`  | Code restructure, no behaviour change |
| `docs`      | Documentation only                    |
| `test`      | Adding or updating tests              |
| `ci`        | CI/CD workflow changes                |

**Examples:**

```
feat(auth): add biometric login support
fix(controls): save button not updating lock
```

### PR Body Template

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

### Rules

- All CI checks must pass before requesting review
- Delete the branch after merging

---

## Branch Naming

### From a GitHub Issue (preferred)

Use the **"Create a branch"** button on the issue page. GitHub generates the branch name automatically:

```
<issue-number>-<issue-title-as-slug>
```

**Examples:**
```
42-feat-add-daily-screen-time-widget
87-bug-login-otp-not-received-on-gmail
103-docs-missing-schedule-api-response-example
```

- Always change the base branch to `develop` in the dialog before creating — GitHub defaults to `main`

### Without a Linked Issue

If the work has no associated issue, use the manual convention:

```
<initials>/<type>/<short-description>
```

**Examples:**
```
jc/feat/onboarding-screen
ab/fix/login-token-refresh
jc/docs/api-endpoint-reference
```

- Initials: 2–3 lowercase letters, agreed with the team
- Use lowercase and hyphens only
- Branch off `develop`, never off `main`
