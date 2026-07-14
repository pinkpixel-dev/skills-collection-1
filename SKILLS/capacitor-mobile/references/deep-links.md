# Deep Links / Universal Links / App Links

Two flavors, pick based on the use case:

- **Custom URL scheme** (`myapp://product/123`) — quick to set up, no domain verification needed, but any app can register the same scheme (mild security/collision risk), and it doesn't gracefully fall back to a website if the app isn't installed.
- **Universal Links (iOS) / App Links (Android)** — real `https://yourdomain.com/...` URLs that open the app if installed and the website otherwise. Requires domain ownership + verification files, but is the right choice for anything user-facing (email links, marketing, OAuth redirects, shared content links) since it's one URL that works everywhere. **This is almost always the one to use for a real product.**

## No extra plugin needed for the JS side

Handling the incoming URL uses the core `@capacitor/app` plugin (already a dependency of every Capacitor project):

```ts
import { App } from '@capacitor/app';

App.addListener('appUrlOpen', (event) => {
  const url = new URL(event.url);
  // route based on url.pathname, same as you'd handle web routing
  router.push(url.pathname + url.search);
});
```

If your mobile app and web app share the same routes (common when it's the same React codebase), this is a straightforward redirect into your existing router — no separate route table to maintain.

## iOS setup (Universal Links)

1. Host an `apple-app-site-association` (AASA) file at `https://yourdomain.com/.well-known/apple-app-site-association` (no file extension, served with `content-type: application/json`, **no redirects** on that path — a domain redirect breaks verification).
   ```json
   {
     "applinks": {
       "apps": [],
       "details": [
         { "appID": "TEAMID.com.yourcompany.yourapp", "paths": ["/product/*", "/user/*", "NOT /api/*"] }
       ]
     }
   }
   ```
2. In Xcode: Signing & Capabilities → **+ Capability** → **Associated Domains** → add `applinks:yourdomain.com` (and `applinks:www.yourdomain.com` if applicable — main domain and `www` are not interchangeable, list both if you use both).
3. Apple fetches the AASA through their own CDN at **install time**, not live — changes can take up to 24h to propagate, and testing by typing the URL directly in Safari's address bar does *not* trigger the interception (only tapping a link from another app — Mail, Messages, etc. — does).

Debug: `codesign -d --entitlements - App.app | grep associated-domains` to confirm the entitlement is actually compiled in; `xcrun simctl erase all` to reset the simulator's Universal Links cache when testing changes.

## Android setup (App Links)

1. Generate a keystore if you don't have one, then get its SHA256 fingerprint:
   ```bash
   keytool -list -v -keystore my-release-key.keystore
   ```
   If you use **Play App Signing** (default for new apps), the fingerprint Google actually uses at runtime is the one in Play Console → App Signing, not your local upload keystore's — this mismatch causes the large majority of "Android App Links not working" reports, so pull the fingerprint from Play Console once the app has App Signing enabled.
2. Host an `assetlinks.json` at `https://yourdomain.com/.well-known/assetlinks.json`:
   ```json
   [{
     "relation": ["delegate_permission/common.handle_all_urls"],
     "target": { "namespace": "android_app", "package_name": "com.yourcompany.yourapp", "sha256_cert_fingerprints": ["AA:BB:CC:..."] }
   }]
   ```
3. Add an intent filter to `android/app/src/main/AndroidManifest.xml`:
   ```xml
   <intent-filter android:autoVerify="true">
     <action android:name="android.intent.action.VIEW" />
     <category android:name="android.intent.category.DEFAULT" />
     <category android:name="android.intent.category.BROWSABLE" />
     <data android:scheme="https" android:host="yourdomain.com" />
   </intent-filter>
   ```

Debug: `curl https://yourdomain.com/.well-known/assetlinks.json`, Google's [Digital Asset Links validator](https://developers.google.com/digital-asset-links/tools/generator), and `adb shell pm get-app-links com.yourcompany.yourapp` on a real device.

## Using deep links for OAuth/auth redirects

Same `appUrlOpen` listener, just treat a specific path as the OAuth callback:

```ts
App.addListener('appUrlOpen', async (event) => {
  const url = new URL(event.url);
  if (url.pathname === '/oauth/callback') {
    const code = url.searchParams.get('code');
    const state = url.searchParams.get('state');
    if (code && validateState(state)) {
      await exchangeCodeForToken(code); // then persist via secure storage — see storage-and-auth.md
      router.push('/home');
    }
  }
});
```
Register your custom redirect URI (either the universal link path or a custom scheme, per what your OAuth provider supports) in the provider's dashboard exactly matching what's declared natively.

## Third-party deep linking services (Branch, etc.)

Worth it if you need **deferred deep linking** (a link tapped before the app is installed still routes correctly after install + first open) or cross-platform attribution analytics. Overkill if you just need "open this URL, go to this screen" — the built-in `@capacitor/app` approach above covers that without a third-party dependency.
