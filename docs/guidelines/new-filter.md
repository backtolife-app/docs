# New Filter Checklist

Everything a new content filter touches, across the four repositories, in the order to do it. Written from the last filter added end to end, Hide Suggested Videos (`youtube_suggested`, user-app #353, 2026-10-16). Copy the list into the issue or a scratch note and tick as you go: a box left unticked is a place the filter silently does not reach.

---

## 0. Decide before writing code

- [ ] **Name** as the person sees it, in EN, ES and IT, worded like its siblings (`Hide …` / `Ocultar …` / `Nascondi …`). The user app capitalises every word (`Ocultar Inicio`); the parent app keeps lower case after the verb (`Ocultar inicio`).
- [ ] **Flag key** `platform_thing` in snake case (`youtube_suggested`). Never a camelCase key: the two that exist (`instagram_likeComments`, `instagramDmOnly`) are legacy contract.
- [ ] **Provider key** in the user app, prefixed per platform (`ytHideSuggested`, `igHide…`, `twHide…`, `tkHide…`), shipped prefixed, no legacy key to migrate.
- [ ] **Item filter or surface filter** (user-app hard rule 10). An item filter removes entries from a list the site fetches: one rule row in the platform's `feedItemFilter` behind a `window.__btl…` flag, failing open. A surface filter removes a part of the interface: a stylesheet or DOM script. Never a new `MutationObserver` or label-text script for an item filter. Exception: Facebook has no data pass (the mobile site is a server-rendered component tree, nothing the page fetches as a list), so its feed-unit filters are DOM scripts on the unit container `[data-type="vscroller"] > *`, feed only, with the page's own marker words in the app's languages (user-app#379).
- [ ] **Free or Premium** in the user app. Premium adds the shutoff wiring below.
- [ ] **Default**: off until switched on, on by default, or locked on for everyone. Locked means `kLockedFilterFlags` in the parent app and a constant in the user app's provider.
- [ ] **Recommended setup**: which wizard tier in the parent app, if any (always-on tier, school-days tier, or not included).
- [ ] **Device capture first**: open the site in the app on the emulator or a phone and read the real DOM or payload (chrome://inspect or the devtools socket) before writing a selector or a field name. The published names are often wrong.
- [ ] No legacy mirror: the flag goes in `FLAGS` only. `strict_shedules` was dropped on 2026-09-04.

## 1. Backend first, since the deploy runs the migration

- [ ] Migration adding `boolean('<flag>')->default(false)` to **both** `strict_schedule_filters` and `parent_strict_schedule_filters` (model: `2026_10_16_000001_add_youtube_suggested_filter_column.php`).
- [ ] `app/Models/ScheduleFilters.php`: `FLAGS` (with a comment: what it hides, free or Premium, parent-selectable), `$fillable`, `$casts`.
- [ ] `app/Models/ParentScheduleFilters.php`: the same three lists.
- [ ] `app/Http/Controllers/API/StrictSchedule/StrictScheduleController.php`: `'filters.<flag>' => 'nullable'`.
- [ ] `app/Http/Controllers/API/Parent/ParentChildController.php`: the same line. A flag missing from either controller is dropped silently.
- [ ] `app/Http/Controllers/Web/Backend/RulesController.php` `FILTER_LABELS`: `'<flag>' => 'Platform · Thing'`. It names the flag on the rules page, in the meeting block and in the weekly usage report; the report accepts the flag through `FLAGS` with no other change.
- [ ] `resources/views/backend/layouts/strict/show.blade.php`: a row in the platform's group.
- [ ] Tests: a case in `tests/Feature/ChildOwnConfigTest.php` and in `tests/Feature/ParentOverlayScheduleTest.php` (flag saved, returned, accepted in `always_on`). Full suite in the container: `docker compose exec laravel.test php artisan test --compact`.
- [ ] Local dev database: `docker compose exec laravel.test php artisan migrate`. Never on production, the pipeline migrates.
- [ ] `docs/api/endpoints.md` and `endpoints.es.md`: a row in the content filter table.
- [ ] `CHANGELOG.md` entry.

## 2. User app

Mirror the closest sibling (Hide Home for a Premium YouTube surface filter, Hide AI Posts for an item filter): grep its flag and its provider key, and touch every file that comes up.

**Script**

- [ ] `lib/features/webviews/scripts/<platform>_scripts.dart`. Item filter: a rule in `feedItemFilter` plus the `FlagSeeds` seed and at-start wrappers. Surface filter: a self-invoking idempotent script with an install guard (`window.__btl<Name>Installed`), a stylesheet set through `textContent` (youtube.com enforces Trusted Types, so `innerHTML` throws), a `MutationObserver` batched with `requestAnimationFrame`, and a docblock with the captured selectors, the capture date and `VERIFY ON DEVICE`.

**State and menu**

- [ ] `lib/features/webviews/widget/<platform>_provider.dart`: field, constructor read, setter, and `resetPremiumFeatures` (condition, reset, write) when Premium.
- [ ] `lib/features/webviews/widget/<platform>_toggle.dart`: `FilterToggleRow`, `PremiumFilterToggleRow` or `AlwaysOnFilterRow`, with `lock: strict.lockFor('<flag>')` and `lockReason`, plus the `on…Toggle` callback.
- [ ] `lib/features/webviews/presentation/<platform>_screen.dart`: the `_strict…` field; set from the `LocalStrictMode` cache (two places) and from the backend response (the canonical `filters` object only, never the legacy top-level field); the live injections in both `_applySchedule` paths; the `_surfaceScripts()` entry (surface) or the document-start seed in `_userScripts()` (item); the three `onLoadStop` branches (strict, toggles, resolved always-on); the toggle callback through `_applyFilterToggle`.
- [ ] `lib/features/home/presentation/app_settings_screen.dart`: `row` or `premiumRow`.
- [ ] `lib/features/more/presentation/filters_features_screen.dart`: the guide entry (name, description).

**Schedules and always-on**

- [ ] `lib/features/block/presentation/always_on_screen.dart`: `_FilterDef`.
- [ ] `lib/features/block/presentation/strict_schedule_screen.dart`: field, read from the cache map, read from the response, reset, `_saveLocalCache` map, payload, `ToggleRowData`.
- [ ] `lib/features/block/data/local_strict_mode.dart`: field, constructor default, `buildAt` with `on(filters['<camel>'] == true, '<flag>')`.
- [ ] `lib/features/block/data/rx_get_schedule/rx.dart`: the legacy-fallback `ScheduleFilters(...)` and the cache map entry.
- [ ] `lib/features/block/data/rx_post_strick_schedule/api.dart`: `ScheduleFiltersPayload` field and `toJson`.
- [ ] `lib/features/block/model/get_schedule_model.dart`: `Data` and `ScheduleFilters`: field, constructor, `fromJson`, `toJson`.

**Reporting and Premium**

- [ ] `lib/features/webviews/service/filter_usage_reporter.dart`: provider key → flag.
- [ ] `lib/navigation_screen.dart` `_maybeReportFirstFilter`: add it when the person can switch it on, never a locked one.
- [ ] `lib/features/premium/presentation/premium_shutoff_watcher.dart` `_anyToggleOn`: Premium only.

**Strings, tests, docs**

- [ ] `lib/l10n/app_en.arb`, `app_es.arb`, `app_it.arb`: `filtersHide<Name>` and `filtersFeatures<Platform><Name>Desc`, then `flutter gen-l10n`.
- [ ] Tests: `test/providers/toggle_provider_test.dart` (key list, default, persistence, Premium reset), `test/features/block/local_strict_mode_test.dart` (resolves from always-on, stays off outside a window when set in the Focus flags only), `test/features/webviews/<platform>_scripts_test.dart` (markers, no `innerHTML`, `node --check` on the emitted string, and the Node harness on real payload shapes for an item filter).
- [ ] `docs/features/webview_filters.md` and `.es.md`; `docs/features/overview.md` and `.es.md` (the platform list).
- [ ] `CHANGELOG.md` entry.
- [ ] `flutter analyze --no-fatal-infos --no-fatal-warnings` and the full `flutter test --reporter compact`.
- [ ] Device: inject the emitted script over the devtools socket on the live page before the build, then check the built app with the toggle on and off, inside a Focus window, as an always-on filter, and after a Premium loss.

## 3. Parent app

- [ ] `lib/features/controls/model/schedule_model.dart`: `ScheduleData` and `ScheduleFilters`: field, constructor, `fromJson`.
- [ ] `lib/features/controls/data/rx_put_schedule/api.dart`: `ScheduleFiltersPayload` field as `required`, so a forgotten call site fails to compile, and `toJson`.
- [ ] `lib/features/rules/presentation/rule_always_on_screen.dart` (`_f`), `rule_allday_filters_screen.dart` (`_F`), `rule_strict_schedule_screen.dart` (field, hydrate, payload, `ScheduleFilterToggleData`). Locked filter: `ScheduleFilterToggleData.locked` and `kLockedFilterFlags` in `lib/constants/locked_filters.dart`.
- [ ] `lib/features/settings/presentation/how_app_works_filters_screen.dart`: the guide row.
- [ ] `lib/features/setup_wizard/data/pending_setup.dart`: the focus payload field.
- [ ] Recommended setup, when decided: `lib/features/setup_wizard/data/wizard_filter_keys.dart` (tier list and flag map), `lib/features/setup_wizard/presentation/setup_wizard_screen.dart` (`_filters` default and `_filterLabel`), `test/unit/pending_setup_test.dart`.
- [ ] `lib/l10n/app_en.arb`, `app_es.arb`, `app_it.arb`: `rulesSocialHide<Name>` and `howWorksDesc<Platform><Name>`, then `flutter gen-l10n`.
- [ ] Tests: `test/unit/schedule_model_test.dart` (payload keys, nested parse), `test/unit/put_schedule_rx_test.dart`, `test/unit/pending_setup_test.dart`.
- [ ] `docs/features/overview.md` and `.es.md` (the `ScheduleData` table), `docs/api/models.md` and `.es.md`, `docs/api/endpoints.md` (the PUT example).
- [ ] `CHANGELOG.md` entry; analyze and the full suite.

## 4. Docs repository

- [ ] `docs/database/schema.md` and `schema.es.md`: the column in the platform's row, with the date.

## 5. Issue and board

- [ ] The issue carries a platform label for every repository touched (`backend`, `parent app`, `user app`), the OS label, the task-type label, and Priority and Size on the board.
- [ ] When asked to reformulate the issue, rewrite title and body as a fresh request for the final scope (free or Premium, default, what it hides, parent, report), never as a report of what was built.

## 6. Commit order

One copy-paste block per repository: backend first, since the deploy migrates, then the user app, the parent app and the docs repository.

## 7. After the next build

- [ ] User app: toggle on and off, the lock inside a Focus window, the always-on pill, the parent-set row, the Premium shutoff.
- [ ] Parent app: the row in Always-on, All-day and Focus, the wizard when included, and the guide.
- [ ] Backend admin: the row on the strict page, the label on the rules page and in the next weekly recap.

---

Reference: the commits for `youtube_suggested` on 2026-10-16 in the four repositories, one filter wired end to end.
