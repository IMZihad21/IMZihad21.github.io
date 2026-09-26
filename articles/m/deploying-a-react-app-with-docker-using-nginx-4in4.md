# Deploying a React App with Docker using Nginx

- Canonical URL: https://imzihad21.github.io/articles/a/deploying-a-react-app-with-docker-using-nginx-4in4/
- Source URL: https://dev.to/imzihad21/deploying-a-react-app-with-docker-using-nginx-4in4
- Web View: https://imzihad21.github.io/articles/a/deploying-a-react-app-with-docker-using-nginx-4in4/
- Published: 2025-03-09T08:26:18.000Z
- Modified: 2025-03-09T08:26:18.000Z
- Reading time: 5 minutes
- Tags: docker, react, nginx, devops

## Deploying a React app with Docker using Nginx

Serving single-page React applications in production requires fast static asset delivery, minimal container attack surfaces, and correct client-side routing resolution. Deploying development servers or retaining the Node.js runtime in production containers wastes memory, introduces security vulnerabilities, and leads to runtime routing failures.

A multi-stage Docker build separating the Node.js build toolchain from an unprivileged Nginx static runtime delivers deterministic asset generation, minimal container image footprints, and reliable single-page application route fallback.

### The problem and production context

Single-page applications (SPAs) are compiled into static bundles of HTML, JavaScript, and CSS during development. Deploying these bundles naively creates operational and infrastructure issues.

- **Failure scenario**: A container image includes the complete Node.js engine and development toolchain to run `npm start` or serve static files through an ad-hoc Node process. The image size balloons to over 1 GB, startup latency increases, and memory consumption remains high. Concurrently, when a user accesses a deep route such as `/dashboard/settings` and refreshes the browser, the web server returns an HTTP 404 Not Found error because no physical file matches the client-side URI path.
- **Why default approaches fall short**: Web servers assume URI paths map directly to filesystem assets. Without an explicit fallback to `index.html`, standard servers fail to delegate route handling to the client-side router (such as React Router). Furthermore, single-stage Dockerfiles retain source code, package managers, and build dependencies in production containers.
- **Production impact**: Bloated images saturate container registries, slow down Kubernetes pod scheduling and CI/CD pipelines, and expose vulnerable build-time dependencies to production networks.

### Mental model and core concepts

Building and deploying a React application with Docker and Nginx relies on separation of concerns across build stages and web server fallback routing.

#### 1. Multi-stage Docker build architecture

Multi-stage builds split image creation into distinct environments. The first stage uses a Node.js image to install dependencies and compile JavaScript bundles. The second stage uses an alpine-based Nginx image, copying only the compiled output from the builder stage. This removes the Node.js engine, package managers, and source files from the final container.

#### 2. Deterministic dependency resolution

Using `npm ci` alongside `package.json` and `package-lock.json` ensures deterministic, repeatable installs. Unlike `npm install`, which can mutate lockfiles or install newer patch versions, `npm ci` strictly enforces lockfile consistency and skips unnecessary dependency resolution passes.

#### 3. Layer caching optimization

Docker builds caches per instruction. Copying `package.json` and `package-lock.json` before copying the rest of the application source code allows Docker to cache the `npm ci` layer. Dependency installation is only re-executed when package manifests change, accelerating iterative builds.

#### 4. SPA routing mechanics and try_files fallback

React client-side routers manage navigation dynamically via browser History APIs. Nginx must intercept requests and attempt to serve matching physical files; if no matching file or directory exists, it must fall back to serving `/index.html`:

```nginx
server {
  listen 80;
  server_name _;
  root /usr/share/nginx/html;
  index index.html;

  location / {
    try_files $uri $uri/ /index.html;
  }
}
```

This guarantees that deep links and page refreshes load the client application, allowing the JavaScript router to render the intended view.

#### 5. Runtime foreground daemon execution

Docker containers exit when their primary process finishes. Nginx must run with `daemon off;` in the foreground so the container runtime can monitor lifecycle health and capture standard output logs directly.

### Production implementation

Define the multi-stage Dockerfile and deployment commands.

```dockerfile
FROM node:20-alpine AS builder
WORKDIR /app

COPY package.json package-lock.json ./
RUN npm ci

COPY . .
RUN npm run build

FROM nginx:1.27-alpine
COPY --from=builder /app/build /usr/share/nginx/html

RUN echo 'server { \
    listen 80; \
    server_name _; \
    root /usr/share/nginx/html; \
    index index.html; \
    location / { \
        try_files $uri $uri/ /index.html; \
    } \
}' > /etc/nginx/conf.d/default.conf

EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

Build and run the containerized React application:

```bash
docker build -t react-app .
docker run -d -p 80:80 --name react-app-container react-app
```

Once running, navigation to any sub-route or deep URL refreshes cleanly to `index.html` without returning HTTP 404 errors.

### Architectural trade-offs and edge cases

Choosing a static Nginx container over server-side rendering (SSR) or Node-based serving introduces operational trade-offs.

* **Latency versus consistency**: Serving pre-compiled static files through Nginx provides high throughput, low CPU utilization, and sub-millisecond response times for cached assets. However, runtime environment configuration is frozen at build time; dynamic environment variables must be injected via runtime scripts or API discovery endpoints rather than read directly from Node `process.env`.
* **Failure recovery**: Because Nginx serves static assets directly from memory and local storage, container restart times are near-instantaneous. If an instance crashes, orchestrators like Kubernetes can spin up replacement pods in milliseconds without cold-start compilation overhead.
* **Scale limitations**: Static Nginx serving handles tens of thousands of concurrent connections on minimal compute allocations. However, it lacks native server-side rendering, which may impact search engine optimization (SEO) or initial paint times for data-dense pages. For dynamic SEO requirements, server-side architectures such as Next.js or Remix should be evaluated instead.

### Common anti-patterns and gotchas

* **Using npm install instead of npm ci**: Running `npm install` in Dockerfiles leads to non-deterministic builds across different environments and risks unintended dependency upgrades. Always commit `package-lock.json` and use `npm ci`.
* **Missing try_files fallback**: Serving React SPAs without configuring `try_files $uri $uri/ /index.html;` causes Nginx to return 404 Not Found whenever a user refreshes a nested route.
* **Shipping Node runtime to production**: Retaining Node.js in the production image increases container image size from approximately 25 MB (Nginx Alpine) to over 500 MB to 1 GB, exposing unnecessary security attack surfaces. Always use multi-stage builds to isolate the build environment.
* **Invalidating Docker cache layers early**: Copying application source code before `package.json` invalidates Docker's layer cache on every code change, forcing long dependency installations on every build.
* **Missing .dockerignore file**: Omitting `.dockerignore` sends local `node_modules`, build artifacts, and secrets to the Docker daemon context, degrading build performance and risking data leakage.

### Implementation checklist

1. Add a `.dockerignore` file excluding `node_modules`, `.git`, and local build directories from the build context.
2. Structure the Dockerfile with a Node.js builder stage using `npm ci` and `npm run build`.
3. Configure the final stage using `nginx:alpine` and copy only compiled assets from `/app/build` to `/usr/share/nginx/html`.
4. Configure Nginx with `try_files $uri $uri/ /index.html;` to support client-side routing.
5. Add HTTP caching headers and Gzip or Brotli compression configuration to Nginx for static assets.
6. Configure container health checks verifying HTTP 200 responses on port 80.
7. Integrate the build into automated CI pipelines with container vulnerability scanning.