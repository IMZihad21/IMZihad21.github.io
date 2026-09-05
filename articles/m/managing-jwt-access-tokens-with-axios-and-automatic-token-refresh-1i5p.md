# Managing JWT Access Tokens with Axios and Automatic Token Refresh

- Canonical URL: https://imzihad21.github.io/articles/a/managing-jwt-access-tokens-with-axios-and-automatic-token-refresh-1i5p/
- Source URL: https://dev.to/imzihad21/managing-jwt-access-tokens-with-axios-and-automatic-token-refresh-1i5p
- Web View: https://imzihad21.github.io/articles/a/managing-jwt-access-tokens-with-axios-and-automatic-token-refresh-1i5p/
- Published: 2023-10-22T18:05:23.000Z
- Modified: 2023-10-22T18:05:23.000Z
- Reading time: 3 minutes
- Tags: axios, jwt, react, tokenrefresh

Many developers hit this issue early: access token expires, user is active, and app suddenly starts throwing `401 Unauthorized`. Users do not enjoy random logout moments.

This guide shows a clean Axios setup where we attach JWT automatically and refresh access token in background when needed.

### Why It Matters

- You keep authenticated sessions smooth without forcing frequent login.
- You avoid repeating auth header logic in every API call.
- You handle concurrent failed requests safely during token refresh.
- You reduce auth bugs that appear only under real traffic.

### Core Concepts

#### 1. API Base URL Configuration

Keep backend URL configurable through environment variables and a fallback.

```javascript
const PROD_BASE_URL = "http://localhost:5000";
const BASE_URL = import.meta.env.VITE_BASE_URL ?? PROD_BASE_URL;
```

#### 2. Axios Instance

Create one shared client for your app API calls.

```javascript
import axios from "axios";

const apiClient = axios.create({ baseURL: BASE_URL });
```

#### 3. Request Interceptor for Bearer Token

Attach latest access token before each request.

```javascript
apiClient.interceptors.request.use((config) => {
  const accessToken = getAccessToken();

  if (accessToken) {
    config.headers.Authorization = `Bearer ${accessToken}`;
  }

  return config;
});
```

#### 4. Detect Expired Token on Response

When API returns `401`, start token refresh flow and retry failed requests.

#### 5. Refresh Lock and Queue

Use one refresh process at a time. Queue all failed requests while refresh is running.

#### 6. Retry Original Requests

After refresh success, retry queued requests with new token. If refresh fails, reject and clear auth data.

### Practical Example

Complete interceptor setup with automatic refresh and safe request queueing:

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

The queue here is like a waiting room for requests. Nobody enters production without a valid token badge.

### Common Mistakes

- Reading access token once at startup instead of per request.
- Triggering multiple refresh calls in parallel under load.
- Retrying requests forever because `_retry` flag is missing.
- Using interceptor client for refresh endpoint and causing recursive loops.
- Not clearing tokens when refresh token is invalid or expired.

### Quick Recap

- Use one Axios instance for consistent API behavior.
- Add token in request interceptor from latest storage state.
- Catch `401` in response interceptor.
- Use refresh lock plus queue for concurrent failed requests.
- Retry once with fresh token, otherwise clear auth and fail fast.

### Next Steps

1. Move token storage to secure strategy based on your app architecture.
2. Add refresh-token rotation on backend for stronger security.
3. Add integration tests for concurrent `401` scenarios.
4. Add logout redirect flow when refresh fails.