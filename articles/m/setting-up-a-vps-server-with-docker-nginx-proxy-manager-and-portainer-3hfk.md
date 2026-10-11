# Setting Up a VPS Server with Docker, Nginx Proxy Manager, and Portainer

- Canonical URL: https://imzihad21.github.io/articles/a/setting-up-a-vps-server-with-docker-nginx-proxy-manager-and-portainer-3hfk/
- Source URL: https://dev.to/imzihad21/setting-up-a-vps-server-with-docker-nginx-proxy-manager-and-portainer-3hfk
- Web View: https://imzihad21.github.io/articles/a/setting-up-a-vps-server-with-docker-nginx-proxy-manager-and-portainer-3hfk/
- Published: 2024-11-17T15:42:43.000Z
- Modified: 2024-11-17T15:42:43.000Z
- Reading time: 6 minutes
- Tags: docker, nginx, portainer, linux

## Setting up a VPS server with Docker, Nginx Proxy Manager, and Portainer

Hosting multiple applications directly on a virtual private server without containerization creates dependency conflicts, complex SSL renewal maintenance, and unmonitored process failures across system services. Managing reverse proxies through manual configuration files introduces configuration drift and operational overhead during TLS certificate issuance.

Deploying Docker, Nginx Proxy Manager for automated Let's Encrypt certificate renewal and reverse proxy routing, and Portainer for container lifecycle management establishes an isolated, reproducible hosting platform on a single host.

### The problem and production context

Running multi-tenant applications directly on bare-metal or raw virtual private server operating systems leads to unisolated ports, permission collisions, and fragile proxy routing.

- **Failure scenario**: An application upgrade overwrites shared runtime libraries or port bindings on the host, taking down adjacent running services while manual Nginx configuration errors invalidate Let's Encrypt automated challenge renewals.
- **Why default approaches fall short**: Operating system package managers distribute outdated Docker packages, and manual reverse proxy configuration requires coordinating raw virtual host files and Certbot cron jobs without unified container network discovery.
- **Production impact**: Unencrypted administrative traffic exposes credentials, manual certificate renewal failures lead to production browser security warnings, and lack of visual container health tracking delays incident recovery.

Keeping deployments isolated using containers, simplifying Let's Encrypt SSL management via a web interface, and maintaining immediate operational visibility into container health establishes a clean pattern for hosting multiple independent applications.

### Mental model and core concepts

A containerized VPS infrastructure separates reverse proxy routing, container execution, and lifecycle orchestration across isolated functional boundaries.

#### 1. Official Docker engine lifecycle and non-root execution

Distribution-provided Docker packages often lag behind stable releases and contain mismatched plugin tooling. Installing Docker Engine, Containerd, Buildx, and Compose from official Docker repositories ensures standard socket APIs. Adding the operating user to the `docker` group permits non-root Docker socket execution without elevating user session privileges.

#### 2. External shared bridge network

Docker Compose creates isolated default bridge networks per project. For Nginx Proxy Manager to route ingress HTTP and HTTPS requests to application containers defined in independent Compose stacks, a shared external Docker bridge network (`nginx_proxy`) is required. Containers connected to this external network resolve one another directly through Docker embedded DNS by container name.

#### 3. Edge ingress and Let's Encrypt automation

Nginx Proxy Manager acts as the single edge ingress point binding public ports 80 (HTTP) and 443 (HTTPS). It negotiates incoming TLS sessions, handles ACME challenge validation for automated Let's Encrypt certificate issuance, and proxies decrypted requests across the internal Docker network directly to upstream application ports.

#### 4. Container management and Docker socket governance

Portainer mounts the host UNIX domain socket (`/var/run/docker.sock`) to communicate directly with the local Docker daemon. This grants Portainer administrative control to inspect container status, retrieve stdout and stderr logs, monitor resource consumption, and manage persistent volumes without requiring SSH access.

### Production implementation

The following implementation configures the host operating system, establishes the network fabric, and deploys Nginx Proxy Manager and Portainer.

Update package repositories, purge legacy container packages, and install the official Docker Engine suite on Debian:

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

Configure non-root Docker execution:

```bash
sudo groupadd docker || true
sudo usermod -aG docker $USER
docker --version
```

Create the external bridge network for shared container discovery:

```bash
docker network create nginx_proxy
```

Create the directory structure for Nginx Proxy Manager:

```bash
mkdir -p ~/managers/nginx
cd ~/managers/nginx
```

Define the Nginx Proxy Manager stack in `docker-compose.yml`:

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

Start Nginx Proxy Manager:

```bash
docker compose up -d
```

Access the admin dashboard at `http://<your-vps-ip>:81` using the initial credentials `admin@example.com` and password `changeme`, and update them immediately.

Create the directory structure for Portainer:

```bash
mkdir -p ~/managers/portainer
cd ~/managers/portainer
```

Define the Portainer stack in `docker-compose.yml`:

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

Start Portainer:

```bash
docker compose up -d
```

Complete initial setup at `http://<your-vps-ip>:9000`.

Expose Nginx Proxy Manager over HTTPS through its own dashboard:
- Domain: `npm.example.com`
- Forward Hostname/IP: `nginx`
- Forward Port: `81`
- Enable SSL, Force SSL, and HTTP/2
- Request a Let's Encrypt certificate

Expose Portainer over HTTPS through the proxy host settings:
- Domain: `portainer.example.com`
- Forward Hostname/IP: `portainer`
- Forward Port: `9000`
- Enable SSL, Force SSL, and HTTP/2
- Request a Let's Encrypt certificate

Once verified at `https://npm.example.com` and `https://portainer.example.com`, close public access to host ports 81 and 9000 in your firewall and route all traffic through port 443. To deploy any new service later, attach its container to the `nginx_proxy` network and configure a proxy host in Nginx Proxy Manager pointing to the container name and internal port.

### Architectural trade-offs and edge cases

Centralizing ingress traffic and container management on a single VPS presents operational trade-offs across security boundaries and resource utilization.

* **Latency versus consistency**: Routing all HTTP traffic through a reverse proxy introduces minor proxy latency overhead while providing immediate, centralized SSL/TLS termination and consistent certificate management across all downstream applications.
* **Failure recovery**: If the Nginx Proxy Manager container crashes or encounters an invalid configuration reload, all hosted domains become temporarily unreachable until the container restarts. SQLite state persisted in `./data` ensures routing rules recover cleanly.
* **Scale limitations**: Mounting `/var/run/docker.sock` directly into Portainer grants root-equivalent control over the host Docker daemon. This setup is appropriate for single-node VPS environments but must be restricted in multi-tenant environments where socket isolation is required.

### Common anti-patterns and gotchas

* **Installing outdated distribution packages**: Installing Docker from default distribution repositories rather than the official Docker repository leads to outdated binaries, missing Compose plugins, and incompatible socket APIs. Use the official Docker Debian repository.
* **Leaving default admin credentials active**: Leaving default admin credentials active on Nginx Proxy Manager permits immediate unauthorized takeover. Update the default email and password during the initial login session.
* **Exposing management ports publicly**: Leaving ports 81 and 9000 permanently open to public internet traffic exposes administrative endpoints to credential brute-forcing. Restrict access behind firewall rules and HTTPS reverse proxy routing.
* **Omitting shared network attachments**: Forgetting to attach new service containers to the shared `nginx_proxy` network prevents Nginx from resolving container DNS names, resulting in 502 Bad Gateway responses. Always attach ingress-facing containers to `nginx_proxy`.
* **Missing persistent volume mounts**: Running containers without mounting host volumes for configuration directories leads to total configuration loss whenever containers are recreated or upgraded. Ensure `./data`, `./letsencrypt`, and `portainer_data` mounts are declared.

### Implementation checklist

1. Run system updates on the Debian host and remove legacy container packages.
2. Add the official Docker GPG signing key and package repository.
3. Install `docker-ce`, `docker-ce-cli`, `containerd.io`, `docker-buildx-plugin`, and `docker-compose-plugin`.
4. Create the `docker` user group and assign the local non-root user.
5. Execute `docker network create nginx_proxy` to create the external shared bridge.
6. Create `~/managers/nginx/docker-compose.yml` and deploy Nginx Proxy Manager using `docker compose up -d`.
7. Sign in to `http://<your-vps-ip>:81`, change the default credentials, and configure admin profile details.
8. Create `~/managers/portainer/docker-compose.yml` and deploy Portainer using `docker compose up -d`.
9. Access `http://<your-vps-ip>:9000` and complete the initial Portainer administrator account creation.
10. Add proxy host for `npm.example.com` forwarding to `nginx:81` with SSL enabled and Force SSL active.
11. Add proxy host for `portainer.example.com` forwarding to `portainer:9000` with SSL enabled and Force SSL active.
12. Configure UFW rules to restrict inbound traffic to ports 22, 80, and 443, and set up fail2ban.
13. Schedule automated backups for the Nginx Proxy Manager (`./data`, `./letsencrypt`) and Portainer data volumes.
14. Add a lightweight monitoring stack to track host resource utilization.
15. Set up a deployment pipeline to automate updates for application containers.