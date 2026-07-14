# Push Notifications

## Plugin choice

Two realistic options:

1. **`@capacitor/push-notifications`** (official) — wraps native APNs (iOS) and FCM (Android). Solid for Android; on iOS it hands you the raw native APNs token, not an FCM token, which trips people up if your backend is expecting to send through Firebase.
2. **`@capacitor-firebase/messaging`** (community, `capacitor-firebase` project) — wraps the Firebase SDK directly on both platforms, so you get a unified FCM token for iOS and Android (and web) with no manual APNs↔FCM bridging. This is the more common recommendation for anyone sending pushes via Firebase Cloud Messaging.

If Firebase is already in the stack for anything else (auth, analytics), `@capacitor-firebase/messaging` avoids fighting the "iOS wall" of APNs-token-vs-FCM-token mismatches. If not using Firebase at all, the official plugin + your own APNs/FCM handling is fine.

## iOS requirements (the expensive part)

- Paid Apple Developer Program membership — push notifications capability won't even appear in provisioning profiles on a free account.
- In Xcode: Signing & Capabilities → add **Push Notifications** capability, and **Background Modes** with "Remote notifications" checked.
- Create an APNs Authentication Key (`.p8` file) in the Apple Developer Portal (preferred over legacy certificates — one key works across all your apps and doesn't expire per-environment).
- Upload the `.p8` to Firebase Console → Project Settings → Cloud Messaging (if using FCM), with the correct Key ID and Team ID — mismatches here are the most common "notifications just don't arrive" bug.
- If you add the Push capability *after* generating provisioning profiles, regenerate them — stale profiles silently fail.

## Android

Much simpler — mostly free, no extra native dependency wrangling since `@capacitor/push-notifications` bundles `firebase-messaging` in its own `build.gradle` automatically. Drop `google-services.json` (from Firebase Console) into `android/app/`.

## Basic usage (official plugin)

```ts
import { PushNotifications } from '@capacitor/push-notifications';

const setup = async () => {
  const permStatus = await PushNotifications.requestPermissions();
  if (permStatus.receive === 'granted') {
    await PushNotifications.register();
  }

  PushNotifications.addListener('registration', (token) => {
    // send token.value to your backend
  });
  PushNotifications.addListener('registrationError', (err) => {
    console.error('Push registration error', err);
  });
  PushNotifications.addListener('pushNotificationReceived', (notification) => {
    // foreground notification received
  });
  PushNotifications.addListener('pushNotificationActionPerformed', (action) => {
    // user tapped a notification
  });
};
```

## Foreground presentation (iOS)

By default foreground notifications may not show a banner unless configured:

```ts
// capacitor.config.ts
const config: CapacitorConfig = {
  plugins: {
    PushNotifications: {
      presentationOptions: ['badge', 'sound', 'alert'],
    },
  },
};
```

## Android notification channels

Android O+ (SDK 26+) requires a notification channel; if you don't create one explicitly with `createChannel()` matching the ID your backend sends, the SDK falls back to a default channel — fine for prototyping, but you'll want explicit channels (e.g. "marketing" vs "transactional") before launch so users can mute one without muting all.

## Common gotchas

- Android Doze mode can delay/restrict delivery — send FCM messages as "high priority" to reduce this.
- Test on a real device or physical build outside of being launched directly from Android Studio/Xcode — dev-launched apps sometimes behave differently around push delivery.
- Images in push payloads: Android handles this automatically; iOS needs a **Notification Service Extension** target added in Xcode to display rich images.
- The official `@capacitor/push-notifications` plugin does **not** support iOS silent/background pushes — if you need that, look at `@capacitor-firebase/messaging` or a dedicated background-fetch approach.
