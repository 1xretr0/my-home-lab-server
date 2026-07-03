# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Docker Compose-based home media server for movie discovery, downloading (torrents), organization, and local network streaming.

## Architecture

The stack follows a pipeline: **Search → Download → Organize → Stream**

```
Radarr (movie mgmt) → Prowlarr (indexer search) → qBittorrent (download)
                                                         ↓
Jellyfin (streaming) ← media/movies/ ← Radarr (rename/move)
Bazarr (subtitles) ←──────────────────┘
```

All services run on a shared Docker bridge network (`medianet`) and reference each other by container name. Configuration is externalized to `.env` for portability across hosts.

**Network binding strategy**: Only Jellyfin binds to `0.0.0.0` (LAN-accessible for TVs/phones). All admin UIs (Radarr, Prowlarr, qBittorrent, Bazarr) bind to `127.0.0.1` — accessible only from the host. qBittorrent also exposes its peer/listening port (51413 TCP+UDP) on all interfaces for torrent connectivity.

**Resource limits**: Jellyfin is capped at 3 CPUs / 4GB RAM via deploy limits (tuned for the Ryzen 5 5600 + 32GB host). Other services have no explicit limits.

**Volume mount design**: Radarr mounts the entire `/data` root (movies + downloads) so it can hardlink/move files from downloads into the library without copying. Other services mount only what they need.

## Commands

```bash
# Start all services
docker compose up -d

# Stop all services
docker compose down

# View logs (follow)
docker compose logs -f <service-name>

# Update images and recreate containers
docker compose pull && docker compose up -d

# Check container status
docker compose ps
```

## Configuration

All paths and ports are defined in `.env`. Inter-service communication uses container names as hostnames (e.g., `http://radarr:7878` from Prowlarr).

## Key Constraints

- `.env` must never be committed with real credentials; `.gitignore` covers `credentials.json`, `.env`, `/config`, and `/media`
- When adding new services, place them on the `medianet` network and follow the existing pattern (PUID/PGID/TZ env vars, config volume at `${CONFIG_PATH}/<service>:/config`)
- Admin UIs bind to `127.0.0.1` — only Jellyfin (streaming) and torrent peer ports should be LAN-exposed
- Radarr needs access to both `movies/` and `downloads/` under a single `/data` mount to enable atomic moves
