# Setting Up a Home Media Server with Docker: A Beginner's Guide

- Canonical URL: https://imzihad21.github.io/articles/a/setting-up-a-home-media-server-with-docker-a-beginners-guide-296l/
- Source URL: https://dev.to/imzihad21/setting-up-a-home-media-server-with-docker-a-beginners-guide-296l
- Web View: https://imzihad21.github.io/articles/a/setting-up-a-home-media-server-with-docker-a-beginners-guide-296l/
- Published: 2024-12-30T14:40:04.000Z
- Modified: 2024-12-30T14:40:04.000Z
- Reading time: 2 minutes
- Tags: docker, linux, devops, mediaserver

A home media server lets you centralize movies, TV shows, and downloads in one place and stream them on your own network. Docker makes this setup easier because each tool runs in an isolated, manageable container.

This guide walks through a practical beginner stack using qBittorrent, Prowlarr, Sonarr, Radarr, and Jellyfin.

### Why It Matters

- Centralizes your media library in one system.
- Automates download and library management workflows.
- Keeps services isolated and easier to maintain.
- Scales gradually as your media collection grows.

### Core Concepts

#### 1. Prerequisites

Before starting, make sure you have:

- A Linux-based home server (PC, mini server, or NAS)
- Docker and Docker Compose installed
- Enough storage for your media files
- Basic terminal and networking familiarity

#### 2. Folder Structure

Create a clean project directory so configs and media stay organized.

```bash
mkdir -p ~/media-server/{configs,downloads,movies,tv}
cd ~/media-server
```

#### 3. Compose-Based Service Stack

Use one `docker-compose.yml` for all media services.

```yaml
services:
  qbittorrent:
    image: lscr.io/linuxserver/qbittorrent:latest
    container_name: qbittorrent
    ports:
      - "8080:8080"
    volumes:
      - ./configs/qbittorrent:/config
      - ./downloads:/downloads
    environment:
      - PUID=1000
      - PGID=1000
    restart: unless-stopped

  prowlarr:
    image: lscr.io/linuxserver/prowlarr:latest
    container_name: prowlarr
    ports:
      - "9696:9696"
    volumes:
      - ./configs/prowlarr:/config
    environment:
      - PUID=1000
      - PGID=1000
    restart: unless-stopped

  sonarr:
    image: lscr.io/linuxserver/sonarr:latest
    container_name: sonarr
    ports:
      - "8989:8989"
    volumes:
      - ./configs/sonarr:/config
      - ./downloads:/downloads
      - ./tv:/tv
    environment:
      - PUID=1000
      - PGID=1000
    restart: unless-stopped

  radarr:
    image: lscr.io/linuxserver/radarr:latest
    container_name: radarr
    ports:
      - "7878:7878"
    volumes:
      - ./configs/radarr:/config
      - ./downloads:/downloads
      - ./movies:/movies
    environment:
      - PUID=1000
      - PGID=1000
    restart: unless-stopped

  jellyfin:
    image: lscr.io/linuxserver/jellyfin:latest
    container_name: jellyfin
    ports:
      - "8096:8096"
    volumes:
      - ./configs/jellyfin:/config
      - ./movies:/media/movies
      - ./tv:/media/tv
    environment:
      - PUID=1000
      - PGID=1000
    restart: unless-stopped
```

#### 4. Launch the Stack

Start all services in detached mode.

```bash
docker compose up -d
```

#### 5. Access the Web UIs

Open these URLs in browser:

- qBittorrent: `http://<server-ip>:8080`
- Prowlarr: `http://<server-ip>:9696`
- Sonarr: `http://<server-ip>:8989`
- Radarr: `http://<server-ip>:7878`
- Jellyfin: `http://<server-ip>:8096`

#### 6. Link the Apps Together

Typical integration flow:

- Add indexers in Prowlarr
- Sync indexers to Sonarr/Radarr
- Configure qBittorrent as download client
- Set root media folders in Sonarr/Radarr
- Add movie/TV libraries in Jellyfin

### Practical Example

After first startup, verify containers are healthy:

```bash
docker compose ps
docker compose logs --tail=50
```

If all services are up and directories are mapped correctly, downloads will move into your library automatically. Automation feels like magic, but it is just good wiring.

### Common Mistakes

- Using inconsistent folder paths across services.
- Forgetting to map TV/movie root folders in Sonarr/Radarr.
- Leaving default credentials unchanged.
- Running old container tags without updates.
- Exposing media server directly to internet without reverse proxy and HTTPS.

### Quick Recap

- Build a clean folder structure first.
- Run all media apps with one Compose stack.
- Share download/media paths consistently.
- Integrate Prowlarr, qBittorrent, Sonarr, and Radarr.
- Use Jellyfin as final streaming layer.

### Next Steps

1. Add reverse proxy + HTTPS for secure remote access.
2. Add automated backups for configs and metadata.
3. Add subtitle automation with Bazarr.
4. Add monitoring and update strategy for long-term stability.