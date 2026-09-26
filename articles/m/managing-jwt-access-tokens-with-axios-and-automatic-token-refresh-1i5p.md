# Managing JWT Access Tokens with Axios and Automatic Token Refresh

- Canonical URL: https://imzihad21.github.io/articles/a/managing-jwt-access-tokens-with-axios-and-automatic-token-refresh-1i5p/
- Source URL: https://dev.to/imzihad21/managing-jwt-access-tokens-with-axios-and-automatic-token-refresh-1i5p
- Web View: https://imzihad21.github.io/articles/a/managing-jwt-access-tokens-with-axios-and-automatic-token-refresh-1i5p/
- Published: 2023-10-22T18:05:23.000Z
- Modified: 2023-10-22T18:05:23.000Z
- Reading time: 5 minutes
- Tags: axios, jwt, react, tokenrefresh

## Managing JWT access tokens with Axios and automatic token refresh

Client applications frequently encounter expired access tokens during active user sessions, resulting in unexpected `401 Unauthorized` errors. Handling token refresh gracefully preserves user sessions without disrupting active tasks, dropping ongoing form submissions, or forcing manual logins.

This architecture establishes a centralized Axios interceptor pipeline that automatically injects JWT bearer tokens into outbound requests, traps authentication failures, and serializes concurrent calls through an in-flight token refresh queue. The implementation guarantees that only a single refresh request executes at any given time, pending calls are queued and replayed upon successful renewal, and authentication failures cleanly purge session state.

### The problem and production context

Single-page and mobile applications experience sudden authentication failures when short-lived JWT access tokens expire. When a dashboard dispatches multiple concurrent API requests on mount (such as user profiles, notification counts, and analytics summaries), multiple endpoints return `401 Unauthorized` simultaneously. Without concurrency controls, naive client implementations trigger multiple parallel refresh requests, causing race conditions with backend token rotation mechanisms.

- **Failure scenario**: A user leaves an application tab idle until their 15-minute access token expires. Upon refocusing, five components trigger simultaneous API requests. Each request receives a 401 response and immediately calls the `/api/refresh-token` endpoint. If the backend enforces one-time refresh token rotation, the first call invalidates the refresh token, causing subsequent concurrent refresh calls to fail and abruptly logging the user out.
- **Why default approaches fall short**: Naive response interceptors attempt to call refresh endpoints using the same Axios instance that intercepts 401 errors, triggering infinite request loops. Furthermore, failing to flag retried requests causes unrecoverable 401 errors to cycle endlessly until the browser hangs or the server rate-limits the client.
- **Production impact**: Uncoordinated refresh attempts corrupt user sessions, trigger premature logouts during active workflows, and create spurious 401 error spikes across frontend monitoring telemetry.

Operational requirements addressed by this design:
- Preserves active user sessions without interrupting workflows with sudden logouts.
- Centralizes authorization header management across all outgoing HTTP calls.
- Coordinates concurrent failed requests during token renewal to prevent race conditions.
- Eliminates silent authentication edge cases before applications face real-world traffic.

### Mental model and core concepts

#### 1. API base URL configuration

Configure the backend API URL dynamically using environment variables with a fallback for local development environments.

```javascript
const PROD_BASE_URL = "http://localhost:5000";
const BASE_URL = import.meta.env.VITE_BASE_URL ?? PROD_BASE_URL;
```

#### 2. Isolated Axios instance creation

Initialize a dedicated Axios client instance to standardize base URLs, request timeouts, and headers while separating general network operations from raw refresh requests.

```javascript
import axios from "axios";

const apiClient = axios.create({ baseURL: BASE_URL });
```

#### 3. Outbound request interceptor for bearer tokens

Attach the latest valid access token dynamically before every outbound request to guarantee that calls dispatched immediately after a token renewal carry current credentials.

```javascript
apiClient.interceptors.request.use((config) => {
  const accessToken = getAccessToken();

  if (accessToken) {
    config.headers.Authorization = `Bearer ${accessToken}`;
  }

  return config;
});
```

#### 4. 401 Unauthorized response detection

Intercept `401 Unauthorized` responses to detect expired access tokens and divert the execution flow into the renewal pipeline.

#### 5. Refresh mutex and asynchronous request queue

Maintain a single active refresh request using a lock boolean (`isRefreshingToken`). Any concurrent unauthorized requests that occur while renewal is in progress are captured as pending Promises and appended to `queuedRequests`.

#### 6. Atomic request replay and session evacuation

Once the refresh request resolves, update authorization headers and replay all queued requests with the newly minted access token. If refresh fails (due to an expired or revoked refresh token), purge authentication credentials from local storage and reject all pending calls.

### Production implementation

The following complete interceptor implementation combines automatic token renewal with a concurrency-safe request queue.

```javascript
import axios from "axios";

const PROD_BASE_URL = "http://localhost:5000";
const BASE_URL = import.meta.env.VITE_BASE_URL ?? PROD_BASE_URL;

const apiClient = axios.create({ baseURL: BASE_URL });

let isRefreshingToken = false;
let queuedRequests = [];

function resolveQueuedRequests(accessToken) {
  queuedRequests.forEach(({ resolve, requestConfig }) => {
    requestConfig.headers.Authorization = `Bearer ${accessToken}`;
    resolve(apiClient(requestConfig));
  });
  queuedRequests = [];
}

function rejectQueuedRequests(error) {
  queuedRequests.forEach(({ reject }) => reject(error));
  queuedRequests = [];
}

apiClient.interceptors.request.use((config) => {
  const accessToken = getAccessToken();

  if (accessToken) {
    config.headers.Authorization = `Bearer ${accessToken}`;
  }

  return config;
});

apiClient.interceptors.response.use(
  (response) => response,
  async (error) => {
    const originalRequest = error?.config;
    const refreshToken = getRefreshToken();
    const isUnauthorized = error?.response?.status === 401;

    if (!isUnauthorized || !originalRequest || originalRequest._retry) {
      return Promise.reject(error);
    }

    if (!refreshToken) {
      removeTokens();
      return Promise.reject(error);
    }

    originalRequest._retry = true;

    if (isRefreshingToken) {
      return new Promise((resolve, reject) => {
        queuedRequests.push({ resolve, reject, requestConfig: originalRequest });
      });
    }

    isRefreshingToken = true;

    try {
      const { data } = await axios.post(`${BASE_URL}/api/refresh-token`, {
        refreshToken,
      });

      const newAccessToken = data?.jwtToken;

      if (!newAccessToken) {
        throw new Error("Missing access token from refresh response");
      }

      setAccessToken(newAccessToken);
      resolveQueuedRequests(newAccessToken);

      originalRequest.headers.Authorization = `Bearer ${newAccessToken}`;
      return apiClient(originalRequest);
    } catch (refreshError) {
      removeTokens();
      rejectQueuedRequests(refreshError);
      return Promise.reject(refreshError);
    } finally {
      isRefreshingToken = false;
    }
  }
);

export default apiClient;
```

This queue holds pending requests while a single refresh call completes, preventing redundant token requests and ensuring each retried call carries valid credentials.

### Architectural trade-offs and edge cases

* **Latency versus consistency**: Queuing concurrent failed requests introduces minor latency for dependent components while the refresh endpoint responds. However, this serialization prevents race conditions and eliminates invalid token rejections across parallel network requests.
* **Failure recovery**: If the refresh token itself has expired or been revoked on the server, the refresh call rejects. The error handler purges client-side tokens and rejects all queued promises, prompting the application state machine to transition to the login screen.
* **Scale limitations**: In environments with hundreds of rapid background polling requests, holding dozens of unresolved requests in an in-memory queue can increase client memory usage. Polling requests should be throttled or paused during active refresh cycles.
* **Loop prevention via retry guards**: Setting `originalRequest._retry = true` prevents infinite retry loops if an endpoint continues returning 401 even after presenting a newly refreshed token.

### Common anti-patterns and gotchas

* **Reading the access token only once at startup**: Caching tokens in closure variables at initialization time prevents interceptors from attaching newly refreshed tokens to subsequent calls. Always resolve tokens dynamically per request.
* **Dispatching multiple refresh requests simultaneously**: Failing to implement a concurrency mutex causes every simultaneous 401 failure to trigger its own refresh call, breaking one-time refresh token rotation policies on the backend.
* **Omitting the retry flag**: Omitting `originalRequest._retry` causes persistent authorization errors (such as lacking permissions on a specific endpoint) to cycle through the refresh interceptor endlessly.
* **Using the intercepted client for refresh calls**: Executing `apiClient.post('/api/refresh-token')` instead of the raw `axios.post` routes the refresh call through the 401 interceptor, causing recursive deadlocks if the refresh call fails.
* **Failing to clear authentication state on rejection**: Leaving expired tokens in client storage after a failed refresh causes subsequent requests to repeat failed refresh attempts continuously.

### Implementation checklist

1. Initialize a dedicated Axios instance with base URL and timeout configurations.
2. Implement dynamic token resolution inside the request interceptor to evaluate fresh tokens on every call.
3. Add a concurrency mutex (`isRefreshingToken`) and request queue array to serialize concurrent 401 responses.
4. Mark original requests with `_retry = true` prior to queueing to prevent circular retry loops.
5. Use raw unintercepted `axios.post` for the token renewal request to avoid interceptor recursion.
6. Implement secure storage patterns suitable for your client architecture, such as HTTP-only cookies.
7. Pair with backend refresh token rotation to mitigate replay vulnerabilities.
8. Write automated unit and integration tests simulating concurrent `401` race conditions.
9. Integrate a global logout redirect event when refresh token validation fails.