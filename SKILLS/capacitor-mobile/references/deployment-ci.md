# App Store / Play Store Deployment & CI/CD

## The two-layer build problem

A Capacitor app has a web build (fast, familiar CI) and a native build (slow, needs macOS for iOS, signing certs, app store tooling) — treat these as separate pipeline stages. Most teams already have JS build/test CI; the native layer is the part that needs new tooling.

## Manual build & release (good to understand even if you later automate it)

### Android
```bash
npm run build
npx cap sync android
cd android && ./gradlew bundleRelease   # produces app-release.aab
```
- Requires a signing keystore (`keytool -genkey -v -keystore my-release-key.keystore -alias ALIAS -keyalg RSA -keysize 2048 -validity 10000`) — back this up somewhere durable, losing it means you can't update the app under the same listing.
- Store credentials in `android/keystore.properties` (gitignored) for CI to reference rather than hardcoding.
- Upload the `.aab` via Play Console → Production (or an internal/alpha/beta track first) → Create new release.
- New apps must submit `.aab` (Android App Bundle), not raw `.apk`.

### iOS
```bash
npm run build
npx cap sync ios
npx cap open ios   # then Archive via Xcode, or use xcodebuild/fastlane for CLI-driven builds
```
- Requires valid provisioning profiles + Distribution certificate.
- Upload to App Store Connect → TestFlight for beta testing → promote to production release.

## Generating icons & splash screens

`@capacitor/assets` (`npx capacitor-assets generate`) generates every required size for both platforms from one source image/logo — do this before first submission; each platform rejects incomplete icon sets.

## CI/CD options, roughly cheapest-to-most-managed

1. **DIY GitHub Actions / GitLab CI**: JS build stage is trivial (any standard Node runner). Native stages need a macOS runner for iOS (GitHub-hosted `macos-latest` works but is slower/pricier than Linux runners) and either a Linux or macOS runner for Android (Linux is fine, faster/cheaper). Store signing secrets (keystore file base64-encoded, Apple certs/profiles, API keys) as CI secrets, never commit them.
2. **fastlane**: the de facto standard CLI for automating iOS/Android signing, building, and store submission — pairs naturally with either DIY CI or Appflow. Worth adopting even without full CI, just to standardize local release builds.
3. **Ionic Appflow**: Capacitor's own paid Mobile DevOps platform — managed iOS/Android build environments (no local Xcode/Android Studio needed), git integration, automated store submission, and "Live Updates" (push JS/HTML/CSS changes to installed apps without an app store review cycle, since only the web layer changes). Worth it mainly if you want to skip owning macOS build infra entirely, or want the live-update capability.
4. **Third-party cloud build/publish services** (Capawesome Cloud Native Builds, Capgo, etc.): similar value prop to Appflow — cloud builds + one-dashboard store publishing + CI hooks via their own CLI. Compare pricing/limits against Appflow if evaluating; the underlying capability (skip local macOS build machine, automate submission) is comparable across these.

## Store review gotchas specific to Capacitor/hybrid apps

- **Subscriptions must use native IAP** for anything unlocked in-app — see `references/subscriptions-iap.md`; this is the single most common rejection reason for hybrid SaaS apps.
- **Privacy manifest (`PrivacyInfo.xcprivacy`)**: required by Apple for apps using certain "required reason" APIs — several common plugins (including `@capacitor/preferences`) need a declared reason code or the build gets flagged in App Store Connect.
- **Product names/prices in IAP UI** must come from the plugin's returned data, not hardcoded strings — mismatches with the actual store listing are a common rejection.
- **Unfinished IAP transactions**: apps that don't call `getUnfinishedTransactions()`/`finishTransaction()` on every launch can get flagged for "broken purchases" during review if the reviewer hits an interrupted purchase state.
- **Associated domains / App Links well-known files** must be live and correctly served (no redirects, correct content-type) *before* submitting if deep linking is part of the review flow.

## PWA distribution note

There's no real "PWA store" review process — distribution is just the web app's own URL plus an install prompt (manifest + service worker). Some app stores (Microsoft Store, some Android flows via Trusted Web Activity) do allow listing a PWA, but that's a separate, optional path from the iOS/Android native builds above, not a requirement.
