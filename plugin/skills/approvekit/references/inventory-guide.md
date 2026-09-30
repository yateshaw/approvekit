# Building the inventory from a repo

The inventory is the input to `audit_app`. Every field maps to something reviewers check. Fill it from files, not from memory. Where the repo has its own backend (a `server/`, `api/` or `supabase/functions` folder next to the app), read it too: scopes, third-party processors and deletion endpoints often live there.

## The schema

| Field | Type | Notes |
|---|---|---|
| `app_name` | string | Display name users see. |
| `developer_name` | string | Legal name that signs the policy. Confirm with the user. |
| `support_email` | string (email) | Public contact for privacy and deletion requests. Confirm with the user. |
| `platforms` | `["ios" \| "android" \| "web"]` | At least one. |
| `has_user_accounts` | boolean | True if users sign up or log in. |
| `account_deletion_in_app` | boolean, optional | True only with working in-app deletion (see §3). |
| `auth_providers` | string[] | e.g. `"Sign in with Apple"`, `"Google"`, `"email and password"`. |
| `data_collected` | `{ type, purpose, shared_with? }[]` | One entry per kind of data (see §5). `shared_with` is a string[] of vendor names. |
| `third_party_sdks` | string[] | Vendor product names (see §6). |
| `google_oauth_scopes` | string[] | Full scope URLs, or `["openid","email","profile"]` for basic sign-in; `[]` if no Google integration. |
| `permissions` | string[] | Raw keys as declared (see §4). |
| `bundle_ids` | `{ ios?, android? }`, optional | Omit for web-only apps. |
| `privacy_policy_url` | string (URL), optional | Omit if none. Do not send `null`. |
| `account_deletion_url` | string (URL), optional | Existing public deletion-request page, if any. |
| `stack` | `"expo" | "react-native" | "flutter" | "ios" | "android" | "web"`, optional | From §1. Expo adds permission strings automatically, so some checks differ. |
| `android_target_sdk` | integer, optional | `targetSdkVersion` from `android/app/build.gradle`; for Expo, the value the current SDK sets (Expo 52+ targets 35) if known. |
| `play_developer_account` | `"personal-new" | "personal-old" | "organization"`, optional | Ask the user: a personal Play account created after 2023-11-13 must run a 12-tester closed test. |
| `takes_payments` | boolean, optional | True only with a payment processor (see §9). |

Omit optional fields you cannot fill. Do not send `null`.

## 1. Detect the stack and platforms

| Evidence | Stack | `platforms` |
|---|---|---|
| `app.json` or `app.config.*` with an `expo` key, `expo` in dependencies | Expo / React Native | `ios`, `android` (add `web` if `expo.web` or `react-dom` is used) |
| `ios/*.xcodeproj`, `Info.plist` | native iOS | `ios` |
| `android/app/build.gradle`, `AndroidManifest.xml` | native Android | `android` |
| `pubspec.yaml` | Flutter | check `ios/` and `android/` folders |
| `next.config.*`, `vite.config.*`, `package.json` with a web framework and no mobile folders | web app | `web` |

A monorepo can contain more than one app. Audit the one the user is publishing; ask if it is not obvious. A web app with `appleWebApp` or a PWA manifest is still `web`; only a store build makes it `ios` or `android`.

## 2. Identifiers (`app_name`, `bundle_ids`)

- Expo: `expo.name`, `expo.ios.bundleIdentifier`, `expo.android.package` in `app.json` / `app.config.*`.
- iOS: `CFBundleDisplayName`, `PRODUCT_BUNDLE_IDENTIFIER` in the `.pbxproj`.
- Android: `applicationId` in `android/app/build.gradle`.
- Flutter: `pubspec.yaml` `name` plus the platform folders above.
- Web: the site title or `metadata.applicationName`; fall back to `package.json` `name`. No `bundle_ids`.

If the folder name and the app name differ, use the app name and mention it in the summary.

## 3. Sign-in (`has_user_accounts`, `auth_providers`, `account_deletion_in_app`)

`has_user_accounts` is true when any of these appear: `@supabase/supabase-js` with `auth.signIn*` calls, `firebase/auth`, `@clerk/*`, `auth0`, `expo-apple-authentication`, `@react-native-google-signin/google-signin`, `next-auth`, `amazon-cognito-*`, `passport`, routes or screens named login, signup or register, a `users` table, password hashing (`bcrypt`, `scrypt`, `argon2`), or session cookies.

`auth_providers`: name each method found, for example `Sign in with Apple`, `Google`, `email and password`, `magic link`, `phone`.

`account_deletion_in_app`: true only if you find working code that deletes the account: a `deleteUser`, `auth.admin.deleteUser`, an RPC or endpoint named `delete_account`/`deleteAccount`/`DELETE /account`, or a settings screen with a "Delete account" action wired to one of those. A "contact us" link is not deletion.

## 4. Permissions (`permissions`)

List the raw keys as the platform declares them, for example `NSCameraUsageDescription` and `android.permission.CAMERA`.

- Expo: `expo.ios.infoPlist` keys ending in `UsageDescription`, `expo.android.permissions`, plus the permissions config plugins add even when they are not written out: `expo-camera` (camera), `expo-location` (fine and coarse location), `expo-contacts` (contacts), `expo-image-picker` (photo library), `expo-local-authentication` (Face ID / biometrics), `expo-audio` or `expo-av` recording (microphone), `expo-notifications` or OneSignal (`POST_NOTIFICATIONS`).
- iOS: every `NS*UsageDescription` key in `Info.plist`.
- Android: every `<uses-permission>` in `AndroidManifest.xml` and `expo.android.permissions`.
- Web: none, unless the app requests browser permissions (camera, geolocation, notifications) in code.

A permission that collects no data (Face ID, biometrics) goes in `permissions` only, not in `data_collected`.

## 5. Data collected (`data_collected`)

Derive entries from permissions, SDKs, auth and the backend. Each entry needs `type` and `purpose`; add `shared_with` when a third party receives the data, including processors that only the backend calls (an AI API, a transcription service, an email provider). If the repo already has a privacy policy, read it and use it as a cross-check: anything it lists that you did not find, and anything you found that it does not list, goes in the summary.

| Evidence | `type` | Typical `purpose` |
|---|---|---|
| any sign-in | `email`, `name` | account login |
| `expo-location`, `ACCESS_*_LOCATION`, `NSLocation*` | `precise location` or `approximate location` | say what the feature does with it |
| `expo-camera`, `NSCameraUsageDescription`, `CAMERA` | `photos or videos` | the feature that uses the camera |
| `expo-image-picker`, `READ_MEDIA_IMAGES`, `NSPhotoLibraryUsageDescription` | `photos` | uploads, avatars, document scanning |
| `expo-contacts`, `READ_CONTACTS`, `NSContactsUsageDescription` | `contacts` | inviting friends, splitting expenses |
| `RECORD_AUDIO`, `NSMicrophoneUsageDescription` | `audio recordings` | voice notes, calls |
| `react-native-purchases` (RevenueCat), `expo-in-app-purchases`, Stripe SDKs | `purchase history` | subscriptions, payments; `shared_with`: RevenueCat, Stripe |
| `react-native-onesignal`, `expo-notifications`, Firebase Messaging | `device push token` | notifications; `shared_with`: OneSignal / Firebase |
| Firebase Analytics, Amplitude, Mixpanel, PostHog, Segment, Google Analytics | `usage data`, `device identifiers` | analytics; `shared_with`: the vendor |
| `@sentry/*`, Crashlytics, Bugsnag | `crash logs`, `device identifiers` | diagnostics; `shared_with`: the vendor |
| AdMob, Meta Audience Network, AppLovin | `advertising identifier` | advertising; `shared_with`: the ad network |
| calls to `api.openai.com`, `api.anthropic.com`, `generativelanguage.googleapis.com`, ElevenLabs, Deepgram (in the app or its backend) | whatever content is sent (`photos of receipts`, `messages`, `voice recordings`) | the AI feature; `shared_with`: the provider |
| user-generated content stored on a backend (Supabase, Firebase, own API) | name the content (`expenses`, `trips`, `notes`) | core app functionality |
| tax IDs, government IDs, bank or card details, health data | name it exactly (`tax ID`, `government ID`) | say why; these are sensitive and reviewers look for them |
| rate limiting or fraud checks by IP | `IP address` | security |

Do not list internal credentials (password hashes, session tokens, invite codes) as collected data.

When the code sends data to an external host you cannot identify, list it with `type: "unknown data sent to <host>"` and ask the user.

## 6. Third-party SDKs (`third_party_sdks`)

List vendors by product name, from `package.json` dependencies, `Podfile`, `build.gradle` or `pubspec.yaml`, plus backend processors. Common mappings:

`@supabase/supabase-js` → Supabase · `firebase`/`@react-native-firebase/*` → Firebase (name the module) · `react-native-purchases` → RevenueCat · `react-native-onesignal` → OneSignal · `@sentry/*` → Sentry · `@stripe/*` → Stripe · `posthog-*` → PostHog · `@amplitude/*` → Amplitude · `mixpanel-*` → Mixpanel · `react-native-google-mobile-ads` → Google AdMob · `@react-native-google-signin/google-signin` → Google Sign-In · `expo-apple-authentication` → Sign in with Apple · `@anthropic-ai/sdk` → Anthropic · `openai` → OpenAI.

Skip framework and UI libraries (React, Expo core modules, navigation, icons, fonts, build-time font loaders).

## 7. Google OAuth scopes (`google_oauth_scopes`)

Search the app and its backend for `googleapis.com/auth/`, `scopes:` or `scope=` near `GoogleSignin.configure`, `google-auth-library`, `googleapis`, `passport-google`, `next-auth` Google provider, or Supabase `signInWithOAuth({ provider: 'google', options: { scopes } })`. Backend-driven consent flows keep the scopes server-side.

- Basic Sign in with Google with no extra scopes → `["openid", "email", "profile"]`.
- Anything under `https://www.googleapis.com/auth/` → the full URL as written.
- No Google integration at all → `[]`.

## 8. Existing policy and contact (`privacy_policy_url`, `support_email`, `developer_name`)

- `privacy_policy_url`: search the repo for `privacy` URLs (settings screens, `app.json`, README, `fastlane/metadata`, store listing files). Only set it if the URL is real and public; otherwise omit it.
- `account_deletion_url`: same search for a public page about deleting the account (often linked from the settings screen or the existing policy).
- `support_email`: `mailto:` links, `support@`, `contact@`, `expo.owner`, `package.json` `author`, an existing policy. Confirm with the user.
- `developer_name`: the legal name that will sign the policy. Check LICENSE, README, an existing policy, or the company in `fastlane/metadata`. If not found, ask; do not guess. If what the user says and what the repo says differ (even in capitalization), ask which is the legal spelling.

## 9. Payments (`takes_payments`)

True only if a payment processor is present: RevenueCat, StoreKit, Google Play Billing, `expo-in-app-purchases`, Stripe, Paddle, Mercado Pago, or a checkout flow. An app that only records money owed between users (an expense splitter, a ledger) is `false`.

## Output

Produce the inventory as JSON matching the schema above. Before calling the tool, show the user a five-line summary (platforms, sign-in, data collected, SDKs and processors, Google scopes). Mark every value you assumed rather than found with "(assumed)" and ask them to confirm `developer_name`, `support_email`, each assumed value and, for Android apps, `play_developer_account`.
