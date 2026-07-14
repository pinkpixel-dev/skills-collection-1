# Native Storage & Secure Token Handling

## Why this needs a native-aware answer (not just localStorage)

Capacitor apps run in a webview, so browser storage APIs (`localStorage`, IndexedDB) technically work — but the OS can evict them under storage pressure, since it treats webview storage the same as any other website's. Fine for cache-able data, not fine for anything you need to survive reliably (auth tokens, subscription entitlement state, user settings).

## The three tiers, pick based on sensitivity + shape of data

### 1. `@capacitor/preferences` (official) — small non-sensitive key/value data
Native `UserDefaults` (iOS) / `SharedPreferences` (Android), falls back to `localStorage` on web/PWA. Good for: theme setting, onboarding-seen flag, non-sensitive user id/display name — **not** for tokens or secrets, it's not encrypted.

```ts
import { Preferences } from '@capacitor/preferences';

await Preferences.set({ key: 'theme', value: 'dark' });
const { value } = await Preferences.get({ key: 'theme' });
await Preferences.remove({ key: 'theme' });
```

String-only values — `JSON.stringify`/`JSON.parse` for objects. iOS privacy manifest note: as of Apple's API-usage-reason requirement, apps using Preferences must declare `NSPrivacyAccessedAPICategoryUserDefaults` with reason code `CA92.1` in a `PrivacyInfo.xcprivacy` file, or App Store Connect will flag the submission.

### 2. Secure storage plugins — sensitive data the app reads in the background (tokens, API keys)
No single official Capacitor plugin here — common picks: `@capawesome-team/capacitor-secure-preferences` or `@aparajita/capacitor-secure-storage`. Both use Android Keystore + iOS Keychain under the hood for real encryption-at-rest, with a `localStorage`-backed web fallback that's explicitly dev-only (never treat the web fallback as secure).

```ts
import { SecurePreferences } from '@capawesome-team/capacitor-secure-preferences';

await SecurePreferences.set({ key: 'refresh_token', value: refreshToken });
const { value } = await SecurePreferences.get({ key: 'refresh_token' });
```

This is the right tier for: OAuth refresh tokens, server-issued API keys, anything the app needs to read automatically without the user unlocking anything.

Android note: exclude the secure-preferences file from cloud backup (`data-extraction-rules.xml`) so encrypted values don't get silently backed up in a form that may not decrypt correctly on restore to a new device.

### 3. Biometric-gated vault (Vault-style plugins) — data that should require explicit user unlock
For anything where access itself should require a fresh biometric/passcode prompt each time (not just "stored encrypted") — password manager-style entries, payment credentials the user re-authenticates to view. Overkill for typical session tokens; reach for this only if the product actually needs an explicit unlock gesture.

### 4. SQLite — structured/relational data, not really an "auth" concern
`@capacitor-community/sqlite` (most maintained option) for offline-first apps needing queries/joins/large record sets. Not the right tool for simple token storage — full relational engine, more setup than the job needs for a handful of keys.

## Recommended default for a subscription SaaS app

- Access token / refresh token / entitlement cache → tier 2 (secure preferences)
- Theme, onboarding flags, non-sensitive UI state → tier 1 (`@capacitor/preferences`)
- Any real offline dataset the app syncs → tier 4 (SQLite), separately from auth

## Biometric unlock for re-authentication

Pair a secure-storage plugin with a biometrics plugin (Face ID / fingerprint) if you want to gate access to stored credentials behind biometric confirmation rather than storing-and-trusting silently — prompt for biometric auth *before* calling `set()` on first login, and again before any sensitive read if the product needs that level of friction.
