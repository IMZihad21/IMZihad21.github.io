# Simplify Email Testing with a Local Papercut SMTP Server Using Docker

- Canonical URL: https://imzihad21.github.io/articles/a/simplify-email-testing-with-a-local-papercut-smtp-server-using-docker-1o6b/
- Source URL: https://dev.to/imzihad21/simplify-email-testing-with-a-local-papercut-smtp-server-using-docker-1o6b
- Web View: https://imzihad21.github.io/articles/a/simplify-email-testing-with-a-local-papercut-smtp-server-using-docker-1o6b/
- Published: 2024-12-30T15:08:05.000Z
- Modified: 2024-12-30T15:08:05.000Z
- Reading time: 2 minutes
- Tags: docker, email, testing, devtools

Testing email locally should be safe and fast. Papercut SMTP captures outgoing emails so you can inspect content, headers, and attachments without sending real messages to real users.

This guide is aligned with the latest Papercut SMTP container docs and updated default ports.

### Why It Matters

- Prevents accidental emails to real recipients.
- Speeds up development and QA for email features.
- Gives a simple web UI to inspect captured messages.
- Works with any app that can send SMTP traffic.

### Core Concepts

#### 1. Updated Default Ports

Current Papercut SMTP Docker defaults are non-privileged:

- Web UI: `8080`
- SMTP: `2525`

This avoids privileged-port issues in Linux containers.

#### 2. Docker Compose Setup

Use the updated port mapping directly:

```yaml
services:
  papercut:
    image: changemakerstudiosus/papercut-smtp:latest
    container_name: papercut_smtp
    ports:
      - "8080:8080"
      - "2525:2525"
    restart: unless-stopped
```

Start service:

```bash
docker compose up -d
```

#### 3. Optional Traditional Host Port Mapping

If your app is locked to classic ports, map host ports to container defaults:

```yaml
services:
  papercut:
    image: changemakerstudiosus/papercut-smtp:latest
    ports:
      - "80:8080"
      - "25:2525"
```

#### 4. Application SMTP Configuration

Point your app to Papercut:

- SMTP Host: `localhost` (or server IP)
- SMTP Port: `2525` (or `25` if remapped)
- Authentication: none
- SSL/TLS: off for local-only flow (unless explicitly configured)

#### 5. Access Web UI

Open:

- `http://localhost:8080` (default mapping)
- `http://localhost` (if `80:8080` mapping used)

#### 6. Development Workflow

Send email from app, inspect in UI, validate content, cleanup inbox, repeat.

### Practical Example

Node.js mail test:

```javascript
import nodemailer from "nodemailer";

const transport = nodemailer.createTransport({
  host: "localhost",
  port: 2525,
  secure: false,
});

await transport.sendMail({
  from: "no-reply@example.local",
  to: "developer@example.local",
  subject: "Papercut SMTP Test",
  text: "Email flow works. Nobody got spammed. Good day.",
});
```

### Common Mistakes

- Using old port assumptions (`80`/`25` inside container) without remapping.
- Pointing local app to production SMTP by mistake.
- Forgetting to verify HTML + plain text + attachment behavior.
- Leaving Papercut exposed publicly on remote hosts.
- Not clearing old test messages between scenarios.

### Quick Recap

- Papercut SMTP is ideal for local email capture.
- Latest container defaults are `8080` (UI) and `2525` (SMTP).
- You can map to `80/25` on host if needed.
- Configure app SMTP to Papercut during development.
- Validate content safely before real provider testing.

### Next Steps

1. Add integration tests for email-trigger endpoints.
2. Add snapshot checks for template output.
3. Add staging verification with real SMTP provider.
4. Add environment guard to block real SMTP in dev mode.

### References

- https://hub.docker.com/r/changemakerstudiosus/papercut-smtp
- https://github.com/ChangemakerStudios/Papercut-SMTP/releases
- https://github.com/ChangemakerStudios/Papercut-SMTP