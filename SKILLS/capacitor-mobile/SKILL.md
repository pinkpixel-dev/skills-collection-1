---
name: capacitor-mobile
description: Complete guide for taking a TypeScript/React (or any modern JS) web app mobile with Capacitor — wrapping it for iOS, Android, and PWA, then adding native features like in-app purchases/subscriptions, push notifications, deep links, secure token storage, and App Store/Play Store deployment with CI/CD. ALWAYS use this skill when the user mentions Capacitor, "make my app mobile," "wrap my webapp for iOS/Android," native app store submission, mobile subscriptions/IAP, RevenueCat, push notifications in a hybrid app, universal links/app links, or Capacitor plugins — even if they only mention one platform or one feature, since these apps almost always need the full stack eventually.
---

# Capacitor Mobile Skill

Capacitor turns an existing web app (React, Vue, vanilla, whatever) into a native-feeling iOS and Android app, while the same codebase can still run as a PWA. It is NOT a rewrite tool — the web app is the source of truth, and Capacitor just wraps it in a native shell and exposes native APIs (via plugins) to JS.

## When this skill applies

Trigger this any time the user is:
- Turning an existing web app into a mobile app ("make it a mobile app", "ship this on iOS/Android")
- Setting up Capacitor from scratch (`npx cap init`, adding platforms)
- Adding subscriptions / in-app purchases / RevenueCat to a Capacitor app
- Adding push notifications, deep links, secure storage, or auth token handling to a Capacitor app
- Prepping a Capacitor app for App Store / Play Store submission or setting up CI/CD for it
- Debugging a Capacitor-specific issue (plugin not found, native build failing, `npx cap sync` issues)

Don't use this for general React/Node work that has nothing to do with the native shell — only reach for these references when the native/mobile layer is actually involved.

## Core mental model

1. **The web app is the app.** Capacitor takes your built web output (`dist/`, `build/`, etc.) and copies it into native iOS/Android projects as a local webview bundle. There's no separate "mobile codebase" — you build once, `npx cap sync` copies assets + updates native deps, then open in Xcode/Android Studio to build the binary.
2. **Plugins are the bridge.** Anything native (camera, push, purchases, secure storage) is exposed to your JS via a plugin with a consistent cross-platform API — call the same TS function, get native behavior on iOS/Android and a graceful web fallback (or no-op) in the browser/PWA.
3. **Three build targets, one codebase**: web (PWA), iOS (Xcode project in `ios/`), Android (Android Studio project in `android/`). Each needs periodic native-side setup (capabilities, permissions, manifests) that lives outside your JS.
4. **PWA is the free one.** If the web app already works, it's already a PWA-capable target with a manifest + service worker — no native tooling needed. iOS/Android require Xcode / Android Studio respectively, plus paid developer accounts to ship ($99/yr Apple, $25 one-time Google).

## Recommended workflow for a new mobile project

1. **Setup** → `references/setup-and-config.md` — installing Capacitor into an existing app, adding iOS/Android/PWA targets, basic config (`capacitor.config.ts`), dev workflow (`npx cap sync`, live reload).
2. **Storage & auth** → `references/storage-and-auth.md` — decide early how tokens/session data persist natively; this affects your auth flow from day one.
3. **Subscriptions/IAP** → `references/subscriptions-iap.md` — since the user's app will have a subscription eventually, decide native-IAP-only vs. RevenueCat now, since retrofitting is more painful than launching correctly the first time.
4. **Push notifications** → `references/push-notifications.md`
5. **Deep links & auth redirects** → `references/deep-links.md`
6. **Deployment & CI/CD** → `references/deployment-ci.md` — App Store/Play Store submission, signing, GitHub Actions pipelines.

Read only the reference file(s) relevant to what the user is doing right now — don't dump the whole skill into context for a narrow question.

## Quick facts worth keeping in your head

- Capacitor is maintained by Ionic; current major version is v8 (v7/v6 still common in older projects — check the user's `package.json` for `@capacitor/core` version before assuming APIs).
- `npx cap sync` = copy web build + install/update native dependencies. Run after every plugin install and every native config change.
- Community plugin ecosystem is fragmented on some features (native purchases, secure storage) — there's no single "official" answer for those, so the reference docs below cover the real tradeoffs rather than pretending there's one blessed plugin.
- Official Capacitor docs: https://capacitorjs.com/docs (versioned — v8 is current, older versions at `/docs/v7/`, `/docs/v6/`, etc.)
