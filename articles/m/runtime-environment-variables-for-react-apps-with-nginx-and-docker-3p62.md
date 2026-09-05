# Runtime Environment Variables for React Apps with Nginx and Docker

- Canonical URL: https://imzihad21.github.io/articles/a/runtime-environment-variables-for-react-apps-with-nginx-and-docker-3p62/
- Source URL: https://dev.to/imzihad21/runtime-environment-variables-for-react-apps-with-nginx-and-docker-3p62
- Web View: https://imzihad21.github.io/articles/a/runtime-environment-variables-for-react-apps-with-nginx-and-docker-3p62/
- Published: 2025-12-25T12:38:37.000Z
- Modified: 2025-12-25T12:38:37.000Z
- Reading time: 2 minutes
- Tags: docker, react, webdev, devops

React frontend env values are usually baked at build time. That means one environment change can force a full rebuild, which is painful when the same image should run in staging, UAT, and production.

This guide shows runtime env injection for a static React app served by Nginx in Docker.

### Why It Matters

- Build once, deploy anywhere.
- Change API endpoints and feature flags without rebuilding image.
- Keeps config external and environment-driven.
- Fits well with Docker, Kubernetes, and CI/CD pipelines.

### Core Concepts

#### 1. Runtime Template File

Create `public/runtime-env.template.js`:

```javascript
window.RUNTIME_ENV = {
  API_URL: "${API_URL}",
  FEATURE_FLAG_ANALYTICS: "${FEATURE_FLAG_ANALYTICS}",
  FEATURE_FLAG_NEW_DASHBOARD: "${FEATURE_FLAG_NEW_DASHBOARD}",
  SENTRY_DSN: "${SENTRY_DSN}",
};
```

#### 2. Load Runtime Config in HTML

Add this in `public/index.html` inside `<head>`:

```html
<script src="/runtime-env.js"></script>
```

#### 3. Consume Config in App Code

```javascript
const config = {
  API_URL: "https://fallback.com",
  FEATURE_FLAG_ANALYTICS: false,
  FEATURE_FLAG_NEW_DASHBOARD: false,
  SENTRY_DSN: "",
  ...(window.RUNTIME_ENV || {}),
};

export default config;
```

#### 4. Docker Multi-Stage Setup

```dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM nginx:alpine
RUN apk add --no-cache gettext
COPY --from=builder /app/build /usr/share/nginx/html
COPY public/runtime-env.template.js /usr/share/nginx/html/runtime-env.template.js
COPY entrypoint.sh /docker-entrypoint.d/40-runtime-env.sh
RUN chmod +x /docker-entrypoint.d/40-runtime-env.sh
```

#### 5. Entrypoint Script for Env Substitution

```bash
#!/bin/sh
set -eu

export API_URL=${API_URL:-}
export FEATURE_FLAG_ANALYTICS=${FEATURE_FLAG_ANALYTICS:-}
export FEATURE_FLAG_NEW_DASHBOARD=${FEATURE_FLAG_NEW_DASHBOARD:-}
export SENTRY_DSN=${SENTRY_DSN:-}

envsubst '${API_URL} ${FEATURE_FLAG_ANALYTICS} ${FEATURE_FLAG_NEW_DASHBOARD} ${SENTRY_DSN}' \
  < /usr/share/nginx/html/runtime-env.template.js \
  > /usr/share/nginx/html/runtime-env.js

rm -f /usr/share/nginx/html/runtime-env.template.js
```

#### 6. Runtime Injection Scope

Expose only explicit variables in template and `envsubst` command.

### Practical Example

Run container with environment values:

```bash
docker run -p 80:80 \
  -e API_URL=https://api.prod.com \
  -e FEATURE_FLAG_ANALYTICS=true \
  -e FEATURE_FLAG_NEW_DASHBOARD=false \
  -e SENTRY_DSN=https://example@sentry.io/123 \
  your-image:latest
```

Container startup generates `runtime-env.js`, and React app reads values immediately. Same image, different environment, zero rebuild drama.

### Common Mistakes

- Trying to read `process.env` directly in browser runtime.
- Forgetting to include `/runtime-env.js` before app bundle.
- Exposing unintended environment variables in template.
- Using shell script that overrides Nginx entrypoint behavior incorrectly.
- Not setting safe fallbacks in frontend config.

### Quick Recap

- Build static app once.
- Generate `runtime-env.js` on container startup.
- Inject only approved environment variables.
- Load config before app bootstraps.
- Reuse same image across all environments.

### Next Steps

1. Add schema validation for `window.RUNTIME_ENV` values.
2. Add environment-specific monitoring tags via runtime config.
3. Add Kubernetes config map/secret mapping for runtime vars.
4. Add startup checks to fail fast on required missing variables.