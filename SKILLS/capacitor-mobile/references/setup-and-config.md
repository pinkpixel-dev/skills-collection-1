# Setup & Platform Config (iOS / Android / PWA)

## Installing into an existing web app

Capacitor drops into any existing modern JS project (Vite, Next.js, CRA, plain webpack — doesn't matter, TypeScript/React included). It doesn't scaffold a new app; it wraps what's there.

```bash
npm install @capacitor/core
npm install -D @capacitor/cli
npx cap init
```

`cap init` asks for app name and package/bundle ID (reverse-domain, e.g. `com.pinkpixel.myapp`). This ID is permanent-ish — changing it later means re-provisioning both stores, so pick it deliberately even in a side project.

This generates `capacitor.config.ts` (or `.json`):

```ts
import type { CapacitorConfig } from '@capacitor/cli';

const config: CapacitorConfig = {
  appId: 'com.pinkpixel.myapp',
  appName: 'MyApp',
  webDir: 'dist', // must match your build output folder
};

export default config;
```

`webDir` has to point at whatever your bundler outputs (`dist` for Vite, `build` for CRA, `out`/`.next` export for Next.js static export — Next.js apps using server features need `next export` or a static-compatible config since Capacitor just serves static files from the device).

## Adding platforms

```bash
npm install @capacitor/ios @capacitor/android
npx cap add ios
npx cap add android
```

This creates real native projects at `ios/` and `android/` — actual Xcode/Android Studio projects, not config stubs. They get committed to git (mostly) and opened directly in the native IDEs when you need platform-specific config.

## The core dev loop

1. Build the web app (`npm run build`)
2. `npx cap sync` — copies the built web assets into both native projects AND updates/installs native dependencies for any Capacitor plugins in `package.json`
3. `npx cap open ios` / `npx cap open android` — opens Xcode / Android Studio
4. Run on simulator/emulator or a real device from there

Run `npx cap sync` after: every web build you want reflected natively, every plugin install/uninstall, and any change to `capacitor.config.ts`.

For faster inner-loop dev, `npx cap run ios --livereload --external` (or the equivalent long-form `npx cap sync` + Xcode run) points the native shell at your local dev server instead of the bundled build, so JS changes hot-reload without a native rebuild.

## Requirements per platform

- **iOS**: macOS + Xcode. Paid Apple Developer Program account ($99/yr) required to run on a physical device or submit to the App Store (simulator works free).
- **Android**: Android Studio (any OS). One-time $25 Google Play Developer registration fee to publish.
- **PWA**: nothing extra — if the web app has a manifest + service worker (Vite PWA plugin, Workbox, etc.) it's already installable. See `references/deployment-ci.md` for store-adjacent PWA distribution notes (there isn't really a "PWA store" — it's installed straight from the browser).

## Configuring your app after the fact

- **App name / package ID**: package ID lives in `android/app/build.gradle` (`applicationId`) and Xcode's bundle identifier setting — changing it after native projects exist means editing both natively, not just in `capacitor.config.ts`.
- **Icons & splash screens**: use `@capacitor/assets` (`npx capacitor-assets generate`) to generate all required sizes for both platforms from one source image, rather than hand-exporting dozens of PNGs.
- **Permissions**: each native plugin that touches a sensitive API (camera, push, IAP) requires platform-specific permission declarations — `Info.plist` entries for iOS, `AndroidManifest.xml` entries for Android, plus (as of iOS 17+ store requirements) `PrivacyInfo.xcxprivacy` "reason" declarations for certain API categories like UserDefaults/Preferences storage. Each plugin's docs specify exactly which entries it needs — don't guess these, they cause store rejections if missing or wrong.

## Using with a UI framework

Capacitor is framework-agnostic. For React/TS specifically: there's no special React wrapper needed for core plugins — you `import { PluginName } from '@capacitor/plugin-name'` and call it like any async JS API. Routing (React Router, TanStack Router, etc.) keeps working as normal inside the webview; the one gotcha is deep link handling needs to hand off into your router manually (see `references/deep-links.md`).
