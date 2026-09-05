# Setting Up a VPS Server with Docker, Nginx Proxy Manager, and Portainer

- Canonical URL: https://imzihad21.github.io/articles/a/setting-up-a-vps-server-with-docker-nginx-proxy-manager-and-portainer-3hfk/
- Source URL: https://dev.to/imzihad21/setting-up-a-vps-server-with-docker-nginx-proxy-manager-and-portainer-3hfk
- Web View: https://imzihad21.github.io/articles/a/setting-up-a-vps-server-with-docker-nginx-proxy-manager-and-portainer-3hfk/
- Published: 2024-11-17T15:42:43.000Z
- Modified: 2024-11-17T15:42:43.000Z
- Reading time: 3 minutes
- Tags: docker, nginx, portainer, linux

Hosting apps on a VPS gets much easier when infrastructure is containerized from day one. With Docker for runtime, Nginx Proxy Manager for reverse proxy + SSL, and Portainer for container management, you get a clean and scalable baseline.

This guide uses a Debian-based server and walks through the full setup step by step.

### Why It Matters

- Keeps deployment predictable with containers.
- Makes domain routing and SSL simpler with Nginx Proxy Manager.
- Gives operational visibility through Portainer UI.
- Builds a reusable foundation for multiple apps on one VPS.

### Core Concepts

#### 1. Install Docker on Debian VPS

Update packages and install Docker using official repository.

```bash
sudo apt update && sudo apt upgrade -y

for pkg in docker.io docker-doc docker-compose podman-docker containerd runc; do
  sudo apt-get remove -y $pkg
done

sudo apt-get update
sudo apt-get install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian $(. /etc/os-release && echo \"$VERSION_CODENAME\") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Configure Docker for non-root usage:

```bash
sudo groupadd docker || true
sudo usermod -aG docker $USER
docker --version
```

#### 2. Create Shared Docker Network

Create one external network for reverse-proxied services.

```bash
docker network create nginx_proxy
```

#### 3. Deploy Nginx Proxy Manager

Create NPM stack and run it with Docker Compose.

```bash
mkdir -p ~/managers/nginx
cd ~/managers/nginx
```

```yaml
services:
  nginx-proxy-manager:
    container_name: nginx
    image: jc21/nginx-proxy-manager:latest
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
      - "81:81"
    volumes:
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
networks:
  default:
    name: nginx_proxy
    external: true
```

```bash
docker compose up -d
```

Default login URL:

```text
http://<your-vps-ip>:81
```

Default credentials:

- Email: `admin@example.com`
- Password: `changeme`

Change these immediately after first login.

#### 4. Deploy Portainer

Create Portainer stack:

```bash
mkdir -p ~/managers/portainer
cd ~/managers/portainer
```

```yaml
services:
  portainer:
    container_name: portainer
    image: portainer/portainer-ce:latest
    restart: unless-stopped
    ports:
      - "9000:9000"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./portainer_data:/data
networks:
  default:
    name: nginx_proxy
    external: true
```

```bash
docker compose up -d
```

Portainer initial URL:

```text
http://<your-vps-ip>:9000
```

#### 5. Expose NPM via Domain and SSL

In NPM UI, create proxy host for NPM dashboard itself:

- Domain: `npm.example.com`
- Forward Hostname/IP: `nginx`
- Forward Port: `81`
- Enable SSL + Force SSL + HTTP/2
- Request Let's Encrypt certificate

#### 6. Expose Portainer via Domain and SSL

Create second proxy host for Portainer:

- Domain: `portainer.example.com`
- Forward Hostname/IP: `portainer`
- Forward Port: `9000`
- Enable SSL + Force SSL + HTTP/2

After validation, you can close direct admin ports and keep only `443` exposed publicly.

### Practical Example

After both proxy hosts are configured:

- `https://npm.example.com` -> Nginx Proxy Manager UI
- `https://portainer.example.com` -> Portainer UI

Now new services can be added behind NPM in minutes, not in three-terminal-debug mode.

### Common Mistakes

- Using Docker from distro packages instead of official repo.
- Keeping default NPM admin credentials.
- Exposing management ports publicly forever.
- Not using a shared Docker network for proxy routing.
- Skipping persistent volume mounts for NPM/Portainer data.

### Quick Recap

- Install Docker from official source.
- Create shared `nginx_proxy` network.
- Deploy NPM and Portainer as separate stacks.
- Configure domain-based SSL reverse proxy entries.
- Restrict insecure management access after setup.

### Next Steps

1. Add UFW hardening rules and fail2ban.
2. Add automated backup for NPM and Portainer volumes.
3. Add monitoring stack for server and container health.
4. Add CI/CD deployment pipeline for app containers.