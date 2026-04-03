# GitHub Project Board

The **BackToLife** project board is the single place to track all work across the three repositories. This guide covers how to use it both as someone creating issues and as a developer picking up tasks.

---

## Board Columns

| Column | Meaning |
|--------|---------|
| **Backlog** | Issues that exist but are not yet prioritised or ready to start |
| **Ready** | Prioritised and fully defined — safe to pick up |
| **In Progress** | Someone is actively working on it |
| **In Review** | A PR is open and waiting for review |
| **Done** | Merged and closed |

---

## For Issue Creators

### 1. Create the issue

Open a new issue using the **Standard Task** template. Fill in the title, bilingual description, technical specs, and all five label groups. See the [Issue & PR Format](issues.md) guide for the full rules.

### 2. Place it on the board

After creating the issue, add it to the **BackToLife** project from the issue's right-hand sidebar under **Projects**. New issues land in **Backlog** by default.

### 3. Move to Ready when it's defined

Only move an issue to **Ready** when:
- The description and technical specs are complete
- Labels (Platform, OS, Priority, Sizing, Task Type) are all set
- It is actually prioritised for the current or upcoming work cycle

Do not leave half-defined issues in Ready — move them back to Backlog if something is missing.

---

## For Developers

### Picking up a task

1. Look in the **Ready** column for your next task — pick by priority label if unsure
2. Assign yourself to the issue
3. Drag the card to **In Progress**
4. Create the branch using the **"Create a branch"** button on the issue page (see [Contributing](contributing.md#branch-naming))

### Opening a PR

1. Push your branch and open a Pull Request
2. In the PR body, include `Closes #<issue-number>` — this links the PR to the issue and will automatically move it to **Done** when merged
3. Drag the issue card to **In Review** (GitHub does not do this automatically)
4. Request a review

### After merge

GitHub automatically closes the linked issue and moves the card to **Done** when the PR is merged, as long as `Closes #<number>` is present in the PR body.

---

## Column Rules at a Glance

| Column | Who moves it there | Condition |
|--------|-------------------|-----------|
| Backlog | Issue creator | Issue created |
| Ready | Issue creator / team lead | Fully defined and prioritised |
| In Progress | Developer | Branch created, work started |
| In Review | Developer | PR opened |
| Done | GitHub (automatic) | PR merged with `Closes #` |
