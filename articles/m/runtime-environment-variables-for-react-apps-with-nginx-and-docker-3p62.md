# Runtime Environment Variables for React Apps with Nginx and Docker

- Canonical URL: https://imzihad21.github.io/articles/a/runtime-environment-variables-for-react-apps-with-nginx-and-docker-3p62/
- Source URL: https://dev.to/imzihad21/runtime-environment-variables-for-react-apps-with-nginx-and-docker-3p62
- Web View: https://imzihad21.github.io/articles/a/runtime-environment-variables-for-react-apps-with-nginx-and-docker-3p62/
- Published: 2025-12-25T12:38:37.000Z
- Modified: 2025-12-25T12:38:37.000Z
- Reading time: 5 minutes
- Tags: docker, react, webdev, devops

## Runtime environment variables for React apps with Nginx and Docker

React environment variables are typically inlined at build time by bundlers like Vite or Webpack. In containerized environments, this model forces a complete container rebuild whenever a configuration variable changes between deployment stages, preventing engineering teams from promoting identical container images across staging, UAT, and production.

Injecting runtime configuration into a static React application served by Nginx solves this operational limitation without requiring server-side rendering. Generating a dynamic browser script during container startup via standard shell substitution gives the application a true build-once, deploy-anywhere deployment lifecycle.

### The problem and production context

Single-page applications execute purely within client browser runtimes and have no native access to the host operating system's environment variables. Bundlers circumvent this by statically replacing references like `import.meta.env.VITE_API_URL` or `process.env.REACT_APP_API_URL` with string literals during the build step.

- **Failure scenario**: A container image built for staging is promoted to production. Because backend API URLs and telemetry keys were baked into the compiled JavaScript chunks during the staging build, the production deployment sends client requests to staging endpoints, causing data contamination and service disruption.
- **Why default approaches fall short**: Maintaining separate container builds per environment violates continuous delivery principles, wastes CI/CD compute time, and invalidates artifact verification. Adopting full server-side rendering (SSR) merely to inject environment variables adds unnecessary server overhead, operational complexity, and maintenance costs.
- **Production impact**: Development teams suffer from long deployment lead times, risk configuration mismatch outages during rollouts, and struggle to manage Kubernetes ConfigMap updates dynamically across environments.

Injecting runtime configuration at container startup addresses these challenges:
- Enables the standard build once, deploy anywhere workflow.
- Allows configuration changes (such as API endpoints or feature toggles) without rebuilding container images.
- Keeps configuration externalized and environment-driven.
- Integrates cleanly with Docker, Kubernetes ConfigMaps, and CI/CD pipelines.

### Mental model and core concepts

#### 1. Runtime template file definition

Create a template file at `public/runtime-env.template.js` containing placeholders for your configuration:

```javascript
window.RUNTIME_ENV = {
  API_URL: "${API_URL}",
  FEATURE_FLAG_ANALYTICS: "${FEATURE_FLAG_ANALYTICS}",
  FEATURE_FLAG_NEW_DASHBOARD: "${FEATURE_FLAG_NEW_DASHBOARD}",
  SENTRY_DSN: "${SENTRY_DSN}",
};
```

#### 2. Loading runtime configuration in HTML head

Reference the generated script inside `<head>` in `public/index.html` before the application bundles load:

```html
<script src="/runtime-env.js"></script>
```

#### 3. Application code consumption with development fallback

Create a configuration module that reads from `window.RUNTIME_ENV` with sensible fallbacks for local development:

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

#### 4. Multi-stage Docker build pipeline

Build the static assets in a build stage, then copy them into an Nginx image alongside an entrypoint script:

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

#### 5. Startup substitution via Nginx entrypoint hooks

The official Nginx image runs scripts in `/docker-entrypoint.d/` on container startup. Use `envsubst` to replace variables and generate the final `runtime-env.js`:

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

#### 6. Variable whitelisting and security boundaries

Only whitelist specific variables within the `envsubst` parameter list to avoid leaking sensitive host environment variables into browser-accessible files.

### Production implementation

The following operational workflow executes the containerized React application with dynamic environment overrides injected at launch.

Run the container by passing environment variables directly:

```bash
docker run -p 80:80 \
  -e API_URL=https://api.prod.com \
  -e FEATURE_FLAG_ANALYTICS=true \
  -e FEATURE_FLAG_NEW_DASHBOARD=false \
  -e SENTRY_DSN=https://example@sentry.io/123 \
  your-image:latest
```

At container launch, the entrypoint script generates `/usr/share/nginx/html/runtime-env.js`. The browser loads the script before booting the bundle, making the active environment values immediately available. This allows you to deploy the exact same image across multiple environments without rebuilding.

### Architectural trade-offs and edge cases

* **Latency versus consistency**: Loading an unbundled `/runtime-env.js` script synchronously inside `<head>` introduces an additional blocking HTTP round-trip prior to bundle parsing. To minimize latency impact, ensure Nginx serves `runtime-env.js` with `Cache-Control: no-cache, no-store, must-revalidate` so updates apply immediately on pod restart without stale client caching.
* **Failure recovery**: If the container starts with missing or invalid environment variables, the shell entrypoint should fail fast if required parameters are missing rather than generating corrupted placeholder strings.
* **Scale limitations**: Because values in `window.RUNTIME_ENV` are exposed publicly to any client inspecting browser memory or network tabs, this architecture must never be used for private secrets, database connection strings, or server private keys. It is strictly limited to public client configuration.

### Common anti-patterns and gotchas

* **Reading process.env in client browser code**: Attempting to read `process.env` dynamically in client-side code after compilation yields `undefined` or runtime errors because the Node.js runtime does not exist in browsers.
* **Placing the runtime-env.js tag after application bundles**: Loading the configuration script asynchronously or placing it after bundle `<script>` tags causes the React application to initialize before `window.RUNTIME_ENV` is populated, falling back to default values.
* **Running unconstrained envsubst without variable filters**: Executing bare `envsubst` without an explicit parameter list replaces any dollar signs present in the template file, corrupting JavaScript syntax. Always pass explicit variable names to `envsubst`.
* **Overriding the default Nginx entrypoint**: Replacing the official Nginx entrypoint binary with a custom entrypoint script often breaks standard configuration parsing and signal handling. Use the `/docker-entrypoint.d/` lifecycle directory instead.
* **Omitting local development fallbacks**: Failing to provide default values when `window.RUNTIME_ENV` is absent breaks local development workflows running outside of Docker containers.

### Implementation checklist

1. Create `public/runtime-env.template.js` containing public configuration keys.
2. Add `<script src="/runtime-env.js"></script>` to `public/index.html` inside `<head>` before bundle scripts.
3. Implement `src/config.js` reading from `window.RUNTIME_ENV` with fallback defaults.
4. Create `entrypoint.sh` executing `envsubst` with explicit variable arguments.
5. Configure multi-stage Dockerfile mounting the entrypoint into `/docker-entrypoint.d/40-runtime-env.sh`.
6. Add schema validation for `window.RUNTIME_ENV` values during app initialization.
7. Tag client-side error telemetry with environment identifiers populated at runtime.
8. Map environment variables to Kubernetes ConfigMaps and Secrets in deployment manifests.
9. Add entrypoint assertions to halt container boot if required environment variables are missing.