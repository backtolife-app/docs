# Product Analytics — PostHog

[PostHog](https://eu.posthog.com/) records what people do inside the two apps: which screens they open and which actions they take. Both apps write to **one PostHog project, hosted in the EU**, so a family's journey reads across them. The admin dashboard keeps the business KPIs; this is the product layer on top (parent-app#108 and its user-app companion; this page is user-app#330).

This page is the reference for every event. **Any change to an event ships with its line here, in the same PR.**

## Rules

| Rule | What it means |
|------|---------------|
| Identity | The person is our **backend user id** (the same id RevenueCat uses), sent with `identify()` after login and on a cold start with a session, and `reset()` on logout. Never an email, never a name. |
| Typed text | No property ever carries text the user typed. The only free text in the system, the "other" answer of the acquisition question, stays on our backend. |
| Session replay | **Off in both apps.** In the user app it stays off for good, because the users are minors. In the parent app it is a separate, later decision, and would need consent, masking, sampling and a retention period. |
| `app` property | Every event carries `app: parent` or `app: user`, registered once at setup. |
| No key, no events | The project key comes from the `POSTHOG_API_KEY` dart-define. A build without it sends nothing. |

The wrappers: [`lib/helpers/analytics.dart`](https://github.com/backtolife-app/user-app/blob/develop/lib/helpers/analytics.dart) in the user app and the same path in the parent app. Event names live there as constants, so a name typo is a compile error.

## Screen views

Screen views are automatic: PostHog's route observer sits on each app's root navigator and reports every named route as a `$screen` event with the route name. The first route, which Flutter names `/`, is reported as `/LoadingScreen`.

**Parent app**

`/LoadingScreen` `/OnboardingFlowScreen` `/LoginScreen` `/SignUpScreen` `/SignUpEmailVerifyScreen` `/EmailVerifyScreen` `/SuccessfulScreen` `/AcquisitionSourceScreen` `/ForgotPasswordScreen` `/PasswordResetScreen` `/PasswordResetSuccessfulScreen` `/NavigationScreen` `/SettingsScreen` `/LinkChildScreen` `/EnterCodeScreen` `/SetupWizardScreen` `/SetParentPasscodeScreen` `/ChildDetailScreen` `/EditProfileScreen` `/ChildErrorScreen` `/ChangePasswordScreen` `/NotificationsSettingsScreen` `/ParentPinScreen` `/ReportABugScreen` `/AboutScreen` `/HowAppWorksScreen` `/HowWorksFiltersScreen` `/HowWorksScheduleScreen` `/HowWorksDailyLimitsScreen` `/HowWorksAlwaysOnScreen` `/HowWorksAlldayScreen` `/HowWorksBedtimeScreen` `/RuleStrictScheduleScreen` `/RuleAlldayFiltersScreen` `/AppBlockingSetupScreen` `/UnblockRequestsScreen`

**User app**

`/LoadingScreen` `/NavigationScreen` `/LoginScreen` `/SignUpScreen` `/SignUpEmailVerifyScreen` `/EmailVerifyScreen` `/SuccessfulScreen` `/ForgotPasswordScreen` `/PasswordResetScreen` `/EditProfileScreen` `/ChangePasswordScreen` `/PermissionsScreen` `/InstagramWebViewScreen` `/TwitterWebViewScreen` `/TikTokWebViewScreen` `/StrickScheduleScreen` `/DailyLimitScreen` `/DesiredUseScreen` `/AlwaysOnScreen` `/BlockScreen` `/LinkToParentScreen` `/LinkedParentsScreen` `/FamilyBlockPasscodeScreen` `/SetupGuideScreen` `/AndroidPermissionWizard` `/FiltersFeaturesScreen` `/HowAppWorksScreen` `/DailyLimitsHelpScreen` `/AlwaysOnHelpScreen` `/BedtimeHelpScreen` `/AlwaysHiddenHelpScreen` `/DigitalCleaningTipsScreen` `/HomeWidgetsScreen` `/InviteFriendsScreen` `/AnalyticsScreen` `/LearnMoreScreen` `/WhatsNewScreen` `/FaqScreen` `/AboutScreen` `/ReportABugScreen`

The YouTube webview is not a named route yet, so its view is not reported; naming it is a one-line change in the user app when wanted.

## Events sent by the apps

Names are `snake_case`. "Parent" and "User" say which app sends the event.

### Onboarding

| Event | App | When | Properties |
|-------|-----|------|------------|
| `onboarding_step_viewed` | Parent, User | A step of a guided flow appears. Parent: the pre-signup onboarding flow and the setup wizard. User: the app tour. | `flow`: `onboarding`, `setup_wizard`, `tour`. `step`: the step's name in code. `index` (onboarding flow only): 0-based position. |
| `onboarding_step_completed` | Parent, User | The user moves forward from a step. Fired for the last step too, so the flow's completion is its last step's completion. | Same as above. |
| `onboarding_skipped` | User | The tour is skipped. | `flow: tour`, `step`: where it was skipped. |
| `acquisition_source_selected` | Parent | The "how did you hear about us" step is answered. Not fired on skip. | `source`: `reddit` `facebook` `instagram` `tiktok` `youtube` `press` `friend` `school` `search` `other`. *Pending: added when the acquisition-source branch merges.* |

### Paywall

The paywall in both apps is RevenueCat's own screen. Plan taps and the purchase start happen inside it, so the apps report that it was shown and how it ended; the purchase itself comes from RevenueCat (next section).

| Event | App | When | Properties |
|-------|-----|------|------------|
| `paywall_shown` | Parent, User | RevenueCat's paywall was presented. Not fired when the user is already entitled and nothing shows. | `trigger`: what opened it. Parent: `premium_lock`. User: `premium_gate`, `family_premium`, `pro_row`, `profile`. |
| `purchase_completed` | Parent, User | The paywall closed with a purchase. | `result: purchased`, `trigger`. |
| `paywall_closed` | Parent, User | The paywall closed without a purchase. | `result`: `cancelled`, `restored`, `error`. `trigger`. |

### Linking a child and a parent

| Event | App | When | Properties |
|-------|-----|------|------------|
| `link_code_generated` | Parent, User | A linking code is produced. Parent: on the add-child screen. User: when the child shows their code, and again on regenerate. | `regenerated: true` on a regenerate (User). |
| `parent_link_created` | User | The child's app learns it now has a parent: the parents fetch turns "has parent" from false to true. This is the event to join the two apps' journeys on. | none |

### Rules, blocks and limits

| Event | App | When | Properties |
|-------|-----|------|------------|
| `block_created` | Parent, User | An app block is turned on. Parent: the per-app toggle in App Blocking setup. User: the child's own self-block. | `where`: `app_blocking_setup` (Parent), `self` (User). |
| `block_removed` | Parent, User | An app block is turned off. | Same as above. |
| `schedule_created` | Parent, User | A rule is saved. Fired at the start of the save, so a failed save still counts as an attempt. | `rule`: Parent `bedtime` `screen_time` `strict_schedule` `allday_filters` `always_on`. User `strict_schedule`. |
| `filters_changed` | Parent | A content filter switch is flipped. | `filter`: the filter key. `enabled`: true/false. `where`: `school`, `night`, `setup_wizard`. |
| `limit_enabled` | User | The child saves their own daily limit. | `enabled`: true when a limit is set, false when cleared. |

### Scroll sessions

| Event | App | When | Properties |
|-------|-----|------|------------|
| `scroll_session_started` | User | A social platform is opened in the app's webview and the scroll probe starts a sitting. | `platform`: `instagram`, `youtube`, `twitter`, `tiktok`. |
| `scroll_session_ended` | User | The sitting ends and is reported to the backend. | `platform`, `seconds`: total time across the sitting's surfaces. |

### Automatic

| Event | App | When |
|-------|-----|------|
| `Application Installed` | both | First launch after install. |
| `Application Opened` | both | Every launch and every return from the background. |
| `Application Backgrounded` | both | The app goes to the background. |
| `$screen` | both | A named route is shown. See the screen tables. |

## Events sent by RevenueCat

RevenueCat's PostHog integration is on in both RevenueCat projects and writes to the same PostHog project, keyed by the same user id. These are the money events; the apps never send them.

| Event | When |
|-------|------|
| `subscription_started` | First paid purchase. |
| `trial_started` | A free trial begins. |
| `trial_converted` | A trial becomes a paid subscription. |
| `trial_cancelled` | A trial is cancelled before converting. |
| `subscription_renewed` | A paid subscription renews. |
| `subscription_cancelled` | Auto-renew is turned off. |
| `subscription_uncancelled` | Auto-renew is turned back on. |
| `subscription_paused` | The subscription is paused (Android). |
| `subscription_expired` | Access ends. |
| `billing_issue` | A charge failed. |
| `product_changed` | The plan changed. |
| `purchase_started` | The user tapped buy inside the paywall (paywall event). |
| `purchase_cancelled` | The purchase was abandoned or failed inside the paywall (paywall event). |

Subscriber attributes are **not** copied to PostHog persons, so the email RevenueCat holds never reaches PostHog.

## Adding an event

1. Add the name as a constant in `AnalyticsEvent` in the app's `analytics.dart`, using the same name in both apps when the step is equivalent.
2. Call `Analytics.track(AnalyticsEvent.<name>, {...})` where the action happens. Do not await it.
3. Put values in properties, never typed text.
4. Add the row to this page in the same PR.
