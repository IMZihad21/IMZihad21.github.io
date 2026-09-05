# Send FCM Event to All Opened Client Windows Instead of Focused (FCM Default)

- Canonical URL: https://imzihad21.github.io/articles/a/send-fcm-event-to-all-opened-client-windows-instead-of-focused-fcm-default-2hdd/
- Source URL: https://dev.to/imzihad21/send-fcm-event-to-all-opened-client-windows-instead-of-focused-fcm-default-2hdd
- Web View: https://imzihad21.github.io/articles/a/send-fcm-event-to-all-opened-client-windows-instead-of-focused-fcm-default-2hdd/
- Published: 2025-03-05T08:13:51.000Z
- Modified: 2025-03-05T08:13:51.000Z
- Reading time: 2 minutes
- Tags: react, firebase, cloudmessaging, webdev

## Send FCM Event to All Opened Client Windows Instead of Focused FCM Default

In many web apps, one user can keep multiple tabs open. Default FCM behavior often updates the active context only, which can leave other tabs stale.

This guide shows a service worker pattern to broadcast one push event to all opened client windows.

### Why It Matters

- Keeps all open tabs synchronized in real time.
- Prevents stale data in background windows.
- Improves UX for dashboards, chat, and collaboration apps.
- Gives full control over push event routing.

### Core Concepts

#### 1. Immediate Service Worker Control

Use `skipWaiting()` and `clients.claim()` to take control quickly.

```javascript
self.addEventListener("install", () => self.skipWaiting());
self.addEventListener("activate", (event) => event.waitUntil(self.clients.claim()));
```

#### 2. Push Payload Broadcast to All Windows

Handle push event in service worker and fan out message via `clients.matchAll()`.

```javascript
self.addEventListener("push", async (event) => {
  const payload = event.data?.json()?.data;

  if (!payload) {
    return;
  }

  const windowClients = await self.clients.matchAll({
    type: "window",
    includeUncontrolled: true,
  });

  windowClients.forEach((client) => {
    client.postMessage({
      type: "push-notification",
      payload,
    });
  });
});
```

#### 3. Notification Click Routing

Handle notification clicks and redirect with encoded payload context.

```javascript
self.addEventListener("notificationclick", (event) => {
  event.notification.close();

  const data = event.notification?.data?.FCM_MSG?.data;

  if (!data) {
    return;
  }

  const encodedPayload = btoa(JSON.stringify(data));
  const targetUrl = `/?pnBgClick=${encodedPayload}`;

  event.waitUntil(self.clients.openWindow(targetUrl));
});
```

#### 4. Firebase Initialization in Service Worker

Initialize Firebase messaging in service worker context.

```javascript
self.importScripts(
  "/firebase/firebase-app-compat.js",
  "/firebase/firebase-messaging-compat.js"
);

firebase.initializeApp({
  apiKey: "YOUR_API_KEY",
  projectId: "YOUR_PROJECT_ID",
  messagingSenderId: "YOUR_SENDER_ID",
  appId: "YOUR_APP_ID",
});

firebase.messaging();
```

#### 5. Client-Side Message Listener

Each tab listens for service worker `message` events.

```javascript
useEffect(() => {
  const handleServiceWorkerMessage = (event) => {
    if (event.data?.type === "push-notification") {
      console.log("Received push payload", event.data.payload);
    }
  };

  navigator.serviceWorker.addEventListener("message", handleServiceWorkerMessage);

  return () => {
    navigator.serviceWorker.removeEventListener("message", handleServiceWorkerMessage);
  };
}, []);
```

#### 6. Dedupe and State Sync

Use message IDs or timestamps to avoid duplicate processing in each tab.

### Practical Example

Flow after push arrives:

1. Service worker receives push payload.
2. Payload is posted to all open window clients.
3. Every tab updates local store/UI.
4. Notification click opens app route with payload context.

Now one event updates every open tab, so users stop seeing different app states in different windows.

### Common Mistakes

- Handling FCM only in focused-tab logic.
- Forgetting `includeUncontrolled: true` for full tab coverage.
- Not checking payload shape before postMessage.
- Missing dedupe strategy for repeated/replayed events.
- Hardcoding fake Firebase config values in production build.

### Quick Recap

- Use service worker push handler as central event entrypoint.
- Broadcast payload to every open window client.
- Add tab-side listener to update app state.
- Handle notification clicks with context-aware routing.
- Add dedupe logic for safe multi-tab processing.

### Next Steps

1. Add payload schema validation before UI updates.
2. Add per-tab dedupe cache keyed by message ID.
3. Add analytics for push delivery across active tabs.
4. Add fallback sync strategy when service worker is unavailable.