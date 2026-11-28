# Send FCM Event to All Opened Client Windows Instead of Focused (FCM Default)

- Canonical URL: https://imzihad21.github.io/articles/a/send-fcm-event-to-all-opened-client-windows-instead-of-focused-fcm-default-2hdd/
- Source URL: https://dev.to/imzihad21/send-fcm-event-to-all-opened-client-windows-instead-of-focused-fcm-default-2hdd
- Web View: https://imzihad21.github.io/articles/a/send-fcm-event-to-all-opened-client-windows-instead-of-focused-fcm-default-2hdd/
- Published: 2025-03-05T08:13:51.000Z
- Modified: 2025-03-05T08:13:51.000Z
- Reading time: 5 minutes
- Tags: react, firebase, cloudmessaging, webdev

## Send FCM events to all open client windows instead of focused FCM default

Users often keep multiple tabs of the same web application open simultaneously across windows and monitors. By default, Firebase Cloud Messaging (FCM) web handling delivers notifications and events to the active or focused tab, leaving background tabs out of sync.

Using a custom service worker to intercept incoming push events and broadcast payloads to every matching client window ensures all open tabs remain synchronized. This decoupling guarantees real-time consistency across tabs without requiring manual browser reloads or uncoordinated WebSocket subscriptions.

### The problem and production context

Standard Firebase Cloud Messaging integrations on the web rely on default browser notifications and foreground message listeners. When a web application spans multiple open tabs, FCM's default behavior prioritizes the currently focused tab.

- **Failure scenario**: An end user collaborates in a customer support dashboard with three tabs open for different tickets. An incoming notification updates status or adds a comment. Only the focused tab receives the FCM event. The two background tabs continue displaying obsolete states until the user manually triggers a full page refresh.
- **Why default approaches fall short**: Client-side SDK listeners (`onMessage`) only trigger inside active, foreground contexts. Default service worker background handlers display native browser notifications but do not automatically fan out raw data payloads across all open window clients belonging to the origin.
- **Production impact**: Users make decisions based on stale data in background tabs, submit conflicting form updates, and miss critical workflow notifications.

Intercepting push events at the service worker level and multicasting to all clients resolves these issues:
- Keeps application state in all open browser tabs synchronized in real time.
- Prevents stale information in background windows.
- Improves user experience for notifications, real-time chats, and collaborative dashboards.
- Gives you complete control over how push payloads are fanned out to client pages.

### Mental model and core concepts

#### 1. Immediate service worker lifecycle control

Claim active clients immediately during service worker activation to begin handling events without requiring a manual page refresh:

```javascript
self.addEventListener("install", () => self.skipWaiting());
self.addEventListener("activate", (event) => event.waitUntil(self.clients.claim()));
```

#### 2. Push payload broadcast across all open window clients

Intercept incoming push events within the service worker and dispatch the payload across all matching windows using `clients.matchAll()`:

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

#### 3. Notification click routing and payload encoding

Handle click interactions on system notifications and redirect the user with encoded payload parameters:

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

#### 4. Firebase service worker context initialization

Initialize Firebase messaging inside the service worker script context:

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

#### 5. Client-side message listener registration

Each open tab registers a listener for `message` events dispatched from the active service worker:

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

#### 6. Deduplication and state synchronization

Because the message reaches every open tab simultaneously, use unique event identifiers or timestamps so each tab updates only its own local state without triggering redundant API calls.

### Production implementation

The complete push delivery and broadcast workflow coordinates the service worker event dispatcher with client tab observers.

When a push event arrives:
1. The service worker captures the push event.
2. The payload is posted to every active window client via `client.postMessage`.
3. Each tab receives the payload and updates its local stores or view state.
4. If the user clicks the operating system notification, the browser navigates to the target route with the payload attached.

The following service worker implementation encapsulates script loading, client claiming, push fanout, and click routing:

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

self.addEventListener("install", () => self.skipWaiting());
self.addEventListener("activate", (event) => event.waitUntil(self.clients.claim()));

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

The client-side component registers the matching receiver in React:

```javascript
import { useEffect } from "react";

export function usePushNotificationReceiver() {
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
}
```

This ensures every open tab reflects the latest state immediately without requiring manual page reloads.

### Architectural trade-offs and edge cases

* **Latency versus consistency**: Broadcasting push payloads directly via `postMessage` delivers updates to open tabs with near-zero latency compared to periodic background polling. However, messages can be dropped if a client tab is suspended or throttled by browser power-saving mechanisms.
* **Failure recovery**: If a client tab is discarded by the browser operating system due to memory pressure, it cannot process incoming `postMessage` events. Applications should re-synchronize state on tab focus change (`visibilitychange` event) to reconcile any missed push events.
* **Scale limitations**: Fanning out large payloads across dozens of open tabs consumes client CPU and IPC bandwidth. Push payloads should remain lightweight notification hints (such as resource IDs and timestamps), prompting clients to fetch granular diffs if needed.

### Common anti-patterns and gotchas

* **Relying solely on client-side foreground listeners**: Registering only `onMessage` ignores background tabs and fails when the active tab loses focus.
* **Omitting includeUncontrolled when querying window clients**: Calling `clients.matchAll({ type: "window" })` without `includeUncontrolled: true` skips newly opened tabs that have not yet established full service worker controller bindings.
* **Failing to validate payload structure before postMessage**: Dispatching unvalidated payloads causes runtime parsing exceptions in client tab listeners. Validate payload schemas before multicasting.
* **Neglecting message deduplication**: When multiple tabs receive the same update, uncoordinated handlers may all trigger duplicate mutation requests to backend APIs. Client tabs must deduplicate events using message IDs.
* **Shipping placeholder Firebase configuration**: Leaving dummy API credentials in the service worker script breaks push registration in production environments.

### Implementation checklist

1. Configure service worker lifecycle hooks with `skipWaiting()` and `clients.claim()`.
2. Initialize Firebase compat scripts inside `firebase-messaging-sw.js`.
3. Intercept `push` events and query all window clients using `matchAll({ type: "window", includeUncontrolled: true })`.
4. Dispatch structured messages via `client.postMessage` to all connected tabs.
5. Handle `notificationclick` events with target URL navigation and payload query parameters.
6. Register `navigator.serviceWorker.addEventListener("message", ...)` in client application components.
7. Add payload schema validation before dispatching UI updates.
8. Maintain an in-memory deduplication set keyed by message ID in each client tab.
9. Track client delivery telemetry to monitor push reliability across active tabs.
10. Implement a polling or fetch fallback for environments where service workers are disabled or unsupported.