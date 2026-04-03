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

### Rule 2 — Bilingual Descriptions

Every issue description must be written in **both English and Spanish**. English first, Spanish immediately after.

```markdown
## Description (EN)

We need to compress user avatars before uploading to S3 to reduce storage costs.

## Descripción (ES)

Necesitamos comprimir los avatares de los usuarios antes de subirlos a S3 para reducir costos de almacenamiento.
```

---

### Rule 3 — Technical Specs

For any task that is not a trivial fix, include a **Technical Specs** section with specific files, endpoints, models, and logic constraints.

```markdown
## Technical Specs

- **File(s):** `lib/features/profile/data/rx_post_avatar/`, `ProfileUpdateController.php`
- **Endpoint / Model / Screen:** `POST /profile-avatar-upload`, `ProfileUpdateController`
- **Logic constraints:** Max size 2 MB after compression, JPEG output only
- **Dependencies or blockers:** Requires `image_compress` package bump to ^6.0
```

---

### Rule 4 — Issue Type

When opening an issue on GitHub, select one **Type**:

| Type        | When to use                                 |
| ----------- | ------------------------------------------- |
| **Bug**     | An unexpected problem or broken behaviour   |
| **Feature** | A request, idea, or new functionality       |
| **Task**    | A specific piece of work — most common type |

---

### Rule 5 — Label Requirements

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

| Label | Effort              |
| ----- | ------------------- |
| `S`   | 1–2 hours           |
| `M`   | 1–3 days            |
| `L`   | A full week or more |
| `XL`  | More than two weeks |

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
