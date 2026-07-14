# Subscriptions & In-App Purchases

## The rule that overrides everything else here

If your subscription unlocks content/features **inside** the iOS or Android app, Apple and Google require it to go through their native in-app purchase systems (App Store / Play Billing) — you legally cannot just charge a credit card via Stripe from inside the app for digital goods/features. Stripe-only billing is fine if:
- the subscription is purchased on the web (outside the app) and the app just checks entitlement status, or
- the app is "reader"-style content tied to a real-world service (rare exception, don't assume it applies)

This is the single most common Capacitor+SaaS gotcha — plan around it before writing checkout code, not after a rejection.

## Three ways to implement it, in order of effort vs. control

### 1. Native IAP directly (most control, most work)
You call a plugin that wraps StoreKit (iOS) and Play Billing (Android) directly, and you own receipt validation and entitlement logic server-side.

Common plugin choices (there's no single "official" Capacitor IAP plugin — pick one):
- `@capgo/native-purchases` (Cap-go) — modern, StoreKit 2 + Play Billing, exposes `Transaction` objects with `receipt`/`jwsRepresentation` for server-side validation
- `@capawesome-team/capacitor-purchases` — similar feature set, StoreKit 2 + Play Billing 8.x

Core flow, same shape regardless of plugin:
```ts
import { NativePurchases } from '@capgo/native-purchases';

const purchase = async (productId: string) => {
  const { transaction } = await NativePurchases.purchaseProduct({ productId });
  // send transaction.receipt (iOS legacy) or transaction.jwsRepresentation (StoreKit 2)
  // to YOUR backend to validate against Apple/Google server APIs before granting access
  await NativePurchases.finishTransaction({ transactionId: transaction.id });
};
```
- **Always** call `getUnfinishedTransactions()` on every app launch and finish/deliver them — interrupted purchases (crash, closed app mid-purchase) leave dangling transactions that must be resolved or Apple will reject the app.
- Provide a "Restore Purchases" button — required by App Store review, calls `syncTransactions()` / `restorePurchases()`.
- Use the **same product ID** for equivalent products on both stores to simplify backend logic — but choose IDs carefully, they often can't be changed or reused once created.
- Validate receipts server-side, never trust the client-reported "success" alone.

### 2. RevenueCat (`@revenuecat/purchases-capacitor`) — recommended default for most solo/small-team apps
Wraps StoreKit + Play Billing + adds a backend that handles receipt validation, renewal/cancellation webhooks, and cross-platform subscription-status tracking, so you skip writing your own validation server.

```bash
npm install @revenuecat/purchases-capacitor
npx cap sync
```
```ts
import { Purchases } from '@revenuecat/purchases-capacitor';

await Purchases.configure({ apiKey: 'YOUR_REVENUECAT_KEY' });

const offerings = await Purchases.getOfferings();
const { customerInfo } = await Purchases.purchasePackage({ aPackage: offerings.current.availablePackages[0] });
const isSubscribed = customerInfo.entitlements.active['premium'] !== undefined;
```
- Enable the "In-App Purchase" capability in Xcode (Signing & Capabilities) — RevenueCat doesn't do this for you.
- Set up entitlements/offerings/products in the RevenueCat dashboard, mirroring what's configured in App Store Connect / Play Console.
- Webhooks let your backend react in real time to renewals/cancellations without polling.
- Use a separate RevenueCat project per app if you ever plan to sell the app — makes account transfer far simpler later.

**When to pick RevenueCat over raw native IAP**: you don't want to build/maintain your own receipt-validation backend, you want cross-platform (iOS/Android/web) entitlement checks in one call, or you want renewal/churn analytics out of the box. Since the user hasn't committed either way yet — RevenueCat is the safer default; it's still "real" native IAP under the hood (same App Store/Play Store rules and cut apply), it just removes the backend-plumbing work.

### 3. Other wrappers (Adapty, Capawesome Purchases, Qonversion, etc.)
Functionally similar to RevenueCat — paywall tooling, receipt validation, cross-platform status. Worth comparing pricing/limits if RevenueCat's free tier doesn't fit, but the integration shape is nearly identical to option 2.

## Testing

- iOS: use StoreKit local testing (Xcode `.storekit` config file, no App Store Connect round-trip needed) for fast iteration, then Sandbox testing against real App Store Connect products before release.
- Android: Play Console's Internal Testing / License Testing tracks — real purchases don't get charged for licensed testers.

## Store requirement to remember

Apple requires displaying the actual product name/price returned by the plugin (`product.title`, `product.priceString`) rather than hardcoded strings in your UI — mismatches are a common rejection reason.
