# Home Lab Media Server

A self-hosted media server for searching, downloading, organizing, and streaming movies on your local network.

## Architecture

```
You (browser) → Radarr → Prowlarr → Indexers (torrent sites)
                  ↓
            qBittorrent (downloads)
                  ↓
            Media Library (/media/movies)
                  ↓
            Jellyfin (streams to TV, browser, desktop)
                  ↓
            Bazarr (adds subtitles)
```

## Services

| Service | URL | Purpose |
|---------|-----|---------|
| Jellyfin | http://localhost:8096 | Stream movies to any device |
| Radarr | http://localhost:7878 | Search and manage movies |
| Prowlarr | http://localhost:9696 | Manage torrent indexers |
| qBittorrent | http://localhost:8080 | Download torrents |
| Bazarr | http://localhost:6767 | Auto-download subtitles |

## Prerequisites

1. **Docker Desktop** installed on your machine
   - Windows: https://docs.docker.com/desktop/install/windows-install/
   - Linux: https://docs.docker.com/engine/install/ubuntu/
   - macOS: https://docs.docker.com/desktop/install/mac-install/

2. **Docker Compose** (included with Docker Desktop)

## Quick Start

```bash
# 1. Clone/navigate to this directory
cd home-lab

# 2. Edit .env to set your timezone and paths (optional)
#    Default paths work out of the box using relative directories

# 3. Start all services
docker compose up -d

# 4. Verify all containers are running
docker compose ps
```

All services will start and be accessible via their web UIs within 30-60 seconds.

## Configuration (Step-by-Step)

Follow these steps **in order** after starting the containers.

### Step 1: Configure qBittorrent

1. Open http://localhost:8080
2. Default credentials: `admin` / check container logs for temporary password:
   ```bash
   docker logs qbittorrent 2>&1 | grep "temporary password"
   ```
3. Go to **Settings (gear icon) → Downloads**:
   - Default Save Path: `/data/downloads/complete`
   - Keep incomplete torrents in: `/data/downloads/incomplete` (enable this checkbox)
4. Go to **Settings → Web UI**:
   - Change the default password to something you'll remember
5. Click **Save**

### Step 2: Configure Prowlarr (Indexer Manager)

1. Open http://localhost:9696
2. On first launch, set up authentication (username/password)
3. Go to **Indexers → Add Indexer**
4. Search for and add public indexers. Recommended starting options:
   - **1337x** — general purpose, good for movies
   - **The Pirate Bay** — large catalog
   - **TorrentGalaxy** — good quality, active community
   - **YTS** — specialized in movies, small file sizes
5. For each indexer, click **Test** to verify it works, then **Save**

### Step 3: Configure Radarr (Movie Manager)

1. Open http://localhost:7878
2. On first launch, set up authentication

#### 3a. Connect Radarr to Prowlarr

The easiest way is from Prowlarr's side:

1. In **Prowlarr**, go to **Settings → Apps → Add**
2. Select **Radarr**
3. Fill in:
   - Prowlarr Server: `http://prowlarr:9696`
   - Radarr Server: `http://radarr:7878`
   - API Key: copy from **Radarr → Settings → General → API Key**
4. Click **Test** then **Save**
5. Prowlarr will automatically sync all indexers to Radarr

#### 3b. Connect Radarr to qBittorrent

1. In **Radarr**, go to **Settings → Download Clients → Add**
2. Select **qBittorrent**
3. Fill in:
   - Host: `qbittorrent`
   - Port: `8080`
   - Username: `admin`
   - Password: (the password you set in Step 1)
4. Click **Test** then **Save**

#### 3c. Configure Root Folder

1. In **Radarr**, go to **Settings → Media Management**
2. Click **Add Root Folder**
3. Set path to: `/data/movies`
4. Click **Save**

### Step 4: Configure Jellyfin (Media Server)

1. Open http://localhost:8096
2. Follow the setup wizard:
   - Set your preferred language
   - Create an admin account
   - **Add Media Library**: click the **+** button
     - Content type: **Movies**
     - Display name: `Movies`
     - Folders: click **+** and add `/data/movies`
   - Configure metadata language (affects poster/description language)
   - Skip the rest or configure remote access as you wish
3. Finish the wizard

### Step 5: Configure Bazarr (Subtitles)

1. Open http://localhost:6767
2. On first launch, set up authentication
3. Go to **Settings → Languages**:
   - Add your preferred subtitle languages (e.g., English, Spanish)
   - Set a default profile
4. Go to **Settings → Providers**:
   - Enable subtitle providers (OpenSubtitles.com is a good start — requires free account)
5. Go to **Settings → Radarr**:
   - Enable Radarr
   - Host: `radarr`
   - Port: `7878`
   - API Key: copy from **Radarr → Settings → General → API Key**
   - Click **Test** then **Save**

## Usage: Downloading Your First Movie

1. Open **Radarr** at http://localhost:7878
2. Click **Add New** (or the + icon)
3. Search for a movie (e.g., "Inception")
4. Select the correct movie from results
5. Set:
   - Root Folder: `/data/movies`
   - Quality Profile: `HD-1080p` (or your preference)
   - Minimum Availability: `Released`
6. Click **Add Movie** — check "Start search for movie" to begin immediately
7. Monitor progress in **Activity** tab (Radarr) or in qBittorrent's UI
8. Once downloaded, the movie appears in **Jellyfin** automatically (may take a few minutes for metadata scan)

## Watching Movies

### Web Browser
Open http://localhost:8096, log in, and play directly.

### Smart TV
1. Install the **Jellyfin** app on your TV (available on most smart TV platforms)
2. When prompted for server address, enter: `http://<your-pc-ip>:8096`
   - Find your PC's IP: run `ipconfig` (Windows) or `ip addr` (Linux)
3. Log in with your Jellyfin credentials

### Desktop (VLC)
1. In VLC: **Media → Open Network Stream**
2. Enter: `http://localhost:8096`
3. Or: use the dedicated Jellyfin Desktop app (https://jellyfin.org/downloads)

### Mobile
Install the Jellyfin app from your device's app store and connect to `http://<your-pc-ip>:8096`

## Managing Services

```bash
# Start all services
docker compose up -d

# Stop all services
docker compose down

# View logs for a specific service
docker compose logs -f radarr

# Restart a specific service
docker compose restart jellyfin

# Update all services to latest versions
docker compose pull && docker compose up -d

# Check service status
docker compose ps
```

## File Structure

```
home-lab/
├── docker-compose.yml      # Service definitions
├── .env                    # Configuration (paths, ports, timezone)
├── config/                 # Persistent service configurations
│   ├── jellyfin/
│   ├── radarr/
│   ├── prowlarr/
│   ├── qbittorrent/
│   └── bazarr/
└── media/                  # Your media files
    ├── movies/             # Organized movie library
    └── downloads/          # Temporary download area
        ├── complete/
        └── incomplete/
```

## Migrating to New Hardware

1. Stop services: `docker compose down`
2. Copy the entire `home-lab/` directory to the new machine
3. Install Docker on the new machine
4. Edit `.env` to update paths if the media drive location changed
5. Run: `docker compose up -d`

All configurations, history, and media are preserved.

## Troubleshooting

**Services can't communicate with each other**
- Ensure all containers are on the same network: `docker network ls` should show `home-lab_medianet`
- Use container names (not localhost) when configuring inter-service connections

**Permission errors on media files**
- Check that PUID/PGID in `.env` matches your host user: run `id` to find your values
- On Windows with Docker Desktop, this is usually not an issue

**qBittorrent login not working**
- Check logs for the temporary password: `docker logs qbittorrent`

**Movie not appearing in Jellyfin**
- Trigger a manual library scan: Jellyfin → Dashboard → Libraries → Scan
- Check the file actually exists in `media/movies/`

**Downloads stuck or slow**
- Verify indexers are working in Prowlarr (test each one)
- Check qBittorrent's connection status (bottom status bar)

## Future Additions

When you're ready to expand, these services drop into the same `docker-compose.yml`:

- **Sonarr** — TV series (same workflow as Radarr)
- **Lidarr** — Music
- **Readarr** — Books/audiobooks
- **Gluetun** — VPN container (route qBittorrent traffic through VPN)
- **Overseerr** — Request system (let family/friends request movies)
- **Flaresolverr** — Bypass Cloudflare protection on indexers
