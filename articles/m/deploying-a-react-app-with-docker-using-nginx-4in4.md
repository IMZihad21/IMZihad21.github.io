# Deploying a React App with Docker using Nginx

- Canonical URL: https://imzihad21.github.io/articles/a/deploying-a-react-app-with-docker-using-nginx-4in4/
- Source URL: https://dev.to/imzihad21/deploying-a-react-app-with-docker-using-nginx-4in4
- Web View: https://imzihad21.github.io/articles/a/deploying-a-react-app-with-docker-using-nginx-4in4/
- Published: 2025-03-09T08:26:18.000Z
- Modified: 2025-03-09T08:26:18.000Z
- Reading time: 2 minutes
- Tags: docker, react, nginx, devops

## Deploying a React App with Docker Using Nginx

For production React deployments, the goal is simple: build once, serve fast, keep image small. A multi-stage Docker build with Nginx is the standard approach for this.

This guide shows a clean setup that separates build tooling from runtime delivery.

### Why It Matters

- Keeps runtime image small and efficient.
- Removes Node build toolchain from production container.
- Serves static assets with high-performance Nginx.
- Supports SPA client-side routing correctly.

### Core Concepts

#### 1. Multi-Stage Docker Strategy

Use one stage for building React assets and one stage for serving with Nginx.

#### 2. Build Stage with Node

Use `node:alpine`, install dependencies, and generate optimized build output.

```dockerfile
FROM node:alpine AS builder
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build
```

#### 3. Runtime Stage with Nginx

Copy build artifacts only into Nginx image.

```dockerfile
FROM nginx:alpine
COPY --from=builder /app/build /usr/share/nginx/html
```

#### 4. SPA Routing Support

Configure Nginx to fallback to `index.html` for unknown routes.

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

#### 5. Container Startup

Expose port and keep Nginx in foreground mode.

```dockerfile
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

#### 6. Deterministic Dependency Install

Use `npm ci` with `package-lock.json` for consistent builds.

### Practical Example

Complete Dockerfile:

```dockerfile
FROM node:alpine AS builder
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=builder /app/build /usr/share/nginx/html
RUN echo 'server { listen 80; server_name _; root /usr/share/nginx/html; index index.html; location / { try_files $uri $uri/ /index.html; } }' > /etc/nginx/conf.d/default.conf

EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

Build and run:

```bash
docker build -t react-app .
docker run -d -p 80:80 --name react-app-container react-app
```

If route refresh does not return 404, your SPA + Nginx configuration is doing exactly what it should.

### Common Mistakes

- Using `npm install --frozen-lockfile` in npm projects instead of `npm ci`.
- Serving SPA without `try_files ... /index.html` fallback.
- Shipping Node runtime to production when only static files are needed.
- Copying full source too early and losing layer-cache efficiency.
- Ignoring `.dockerignore` and bloating build context.

### Quick Recap

- Build React assets in Node stage.
- Serve static output in Nginx stage.
- Keep runtime image minimal and secure.
- Use SPA fallback for client-side routes.
- Use `npm ci` for deterministic dependency installs.

### Next Steps

1. Add gzip/brotli and cache headers in Nginx config.
2. Add health checks in deployment platform.
3. Add CI pipeline for build, scan, and push.
4. Add runtime env injection strategy if needed.