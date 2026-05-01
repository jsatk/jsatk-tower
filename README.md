# jsatk-tower Infrastructure Documentation

> Last updated: May 1, 2026  
> Hardware: Ugreen DXP4800 Plus · OS: Unraid 7.2.4 · Hostname: `jsatk-tower`

---

## Table of Contents

1. [Hardware Overview](#hardware-overview)
2. [Networking Architecture](#networking-architecture)
3. [Docker Container Stack](#docker-container-stack)
4. [Media Automation Pipeline](#media-automation-pipeline)
5. [Plex & Playback](#plex--playback)
6. [Monitoring & Notifications](#monitoring--notifications)
7. [Backup](#backup)
8. [Storage Layout](#storage-layout)
9. [Access Reference](#access-reference)

---

## Hardware Overview

| Component | Detail |
|---|---|
| Device | Ugreen DXP4800 Plus |
| CPU | Intel N100 (with QuickSync hardware transcoding) |
| OS | Unraid 7.2.4 |
| Hostname | `jsatk-tower` |
| LAN IP | `192.168.142.135` |
| Router | AmpliFi Alien, subnet `192.168.142.0/24`, gateway `192.168.142.1` |
| Unraid WebUI | `http://jsatk-tower.local:82` (HTTP port 82, HTTPS port 444) |

### Storage Configuration

The array uses Unraid's parity-based storage. Layout:

| Slot | Size | Drive | Device |
|---|---|---|---|
| Parity | 28TB | Seagate IronWolf Pro | sdd |
| Disk 1 | 28TB | Seagate IronWolf Pro | sdc |
| Disk 2 | 28TB | Seagate IronWolf Pro | sde |
| Disk 3 | 28TB | Seagate IronWolf Pro | sdb |
| Cache | 1TB | Samsung 990 EVO Plus NVMe | — |

Usable capacity is approximately **~84TB** (28TB × 3 data drives). Parity is a dedicated separate drive and does not reduce usable space. Unraid presents all data drives as a unified pool via the FUSE overlay at `/mnt/user/`. The NVMe cache is formatted btrfs and handles cache-first writes for the `data` share.

> **Important:** For hardlinks (required by the *arr stack), use the underlying disk paths like `/mnt/disk1/...` rather than `/mnt/user/`. The FUSE overlay does not support hardlinks across different underlying drives, which silently breaks the *arr stack's move/rename operations.

---

## Networking Architecture

This is the most complex part of the setup. The goal is: services are accessible securely from outside the LAN via HTTPS with real domain names, without exposing everything to the public internet.

### Domain & DNS

- **Registrar:** Namecheap
- **DNS provider:** Cloudflare
- **Primary domain:** `jsatk.us`

Two DNS records drive everything:

| Record | Type | Value | Purpose |
|---|---|---|---|
| `vpn.jsatk.us` | A | `<YOUR_PUBLIC_IP>` (public IP) | WireGuard VPN endpoint, kept current by `cloudflareddns` container |
| `*.jsatk.us` | A | `192.168.142.135` (LAN IP) | Wildcard — resolves all subdomains to the NAS on LAN |
| `seerr.jsatk.us` | A | `<YOUR_PUBLIC_IP>` (public IP) | Publicly accessible Seerr (proxied through Cloudflare) |

The wildcard `*.jsatk.us → LAN IP` means that when you're on LAN (or connected via WireGuard), `sonarr.jsatk.us`, `plex.jsatk.us`, etc. all resolve directly to the NAS. No split-DNS tricks needed.

### SSL Certificate

A wildcard TLS certificate for `*.jsatk.us` is issued by Let's Encrypt and managed by Nginx Proxy Manager using the **Cloudflare DNS-01 challenge**. Because it's a DNS challenge (not HTTP), the NAS doesn't need to be publicly reachable for cert issuance or renewal — Cloudflare handles the challenge token via the API. The cert covers all `*.jsatk.us` subdomains.

### WireGuard VPN

WireGuard is built into Unraid (interface `wg0`) and is used for **remote access to self-hosted services only** — not as a general-purpose traffic tunnel.

| Setting | Value |
|---|---|
| Endpoint | `vpn.jsatk.us:51820` |
| Tunnel network | `10.253.0.0/24` |
| jesse-iphone peer | `10.253.0.2` |
| jesse-mac-studio peer | `10.253.0.3` |
| Peer DNS | `1.1.1.1` |

Port `51820/UDP` is forwarded on the AmpliFi Alien to `192.168.142.135`.

The `cloudflareddns` container monitors the public IP and updates the `vpn.jsatk.us` DNS A record via the Cloudflare API every 300 seconds. This means the VPN endpoint always points at the correct public IP even if the ISP changes it (dynamic IP).

When connected via WireGuard from iPhone or Mac Studio, traffic destined for `*.jsatk.us` resolves to `192.168.142.135` (LAN IP), so all service access goes directly through the encrypted tunnel to the NAS.

**Note:** Tailscale was fully removed from jsatk-tower, Mac Studio, and iPhone. WireGuard replaced it entirely.

### Nginx Proxy Manager (NPM)

NPM (mgutt fork) is the reverse proxy sitting in front of all services. It terminates HTTPS and forwards requests to the appropriate container on the internal `jsatk-net` Docker bridge network.

| Port | Purpose |
|---|---|
| 80 | HTTP (redirects to HTTPS) |
| 81 | NPM admin UI |
| 443 | HTTPS (all proxied services) |

Port `443/TCP` and `32400/TCP` (Plex) are forwarded on the AmpliFi Alien to `192.168.142.135`.

**Access control:** NPM has an access list called "LAN & VPN only" that allows:
- `192.168.142.0/24` (LAN)
- `10.253.0.0/24` (WireGuard tunnel)
- Deny all others

This access list is applied to **all proxy hosts except `seerr` and `plex`**, which are intentionally public.

### What's Public vs. Private

| Service | Public? | Notes |
|---|---|---|
| `seerr.jsatk.us` | ✅ Yes | Cloudflare-proxied, Plex OAuth only (no local sign-in), port 443 forwarded |
| `plex.jsatk.us` | ✅ Yes | Port 32400 forwarded directly |
| Everything else | ❌ No | LAN + WireGuard only via NPM access list |

### Port Forwarding Summary

| Port | Protocol | Forwarded To | Purpose |
|---|---|---|---|
| 443 | TCP | 192.168.142.135 | HTTPS (NPM → all services) |
| 32400 | TCP | 192.168.142.135 | Plex direct |
| 51820 | UDP | 192.168.142.135 | WireGuard VPN |

### Pi-hole (Network-Wide DNS)

Pi-hole runs as a Docker container on the default `bridge` network (not `jsatk-net`) at `192.168.142.135`, with its web UI accessible at `pihole.jsatk.us` via NPM (LAN/VPN only).

| Setting | Value |
|---|---|
| Web UI (host) | `:8155` → `pihole.jsatk.us` |
| DNS port | `53` (TCP+UDP) |
| Upstream DNS | `1.1.1.1` and `1.0.0.1` (Cloudflare) |
| Listening mode | `ALL` interfaces |
| Timezone | `America/Los_Angeles` |
| Conditional forwarding | Enabled — `192.168.142.0/24` → `192.168.142.1` (router) for local hostname resolution |

**Router configuration (AmpliFi Alien):**
- Bypass DNS Cache: **ON**
- Built-in Ad Blocker: **OFF** (Pi-hole handles this)
- WAN DNS Primary: `192.168.142.135` (Pi-hole)
- WAN DNS Secondary: `1.1.1.1` (fallback)

This means all DNS queries on the LAN flow through Pi-hole first. The conditional forwarding entry ensures that local hostnames (e.g. `jsatk-tower.local`) still resolve correctly via the router.

WireGuard peers use `1.1.1.1` as their DNS rather than Pi-hole, so ad blocking does not apply when connected remotely via VPN.

### How a Request Flows (Example: Sonarr from iPhone)

1. iPhone connects to WireGuard → gets tunnel IP `10.253.0.2`
2. Browser navigates to `sonarr.jsatk.us`
3. DNS resolves `*.jsatk.us` → `192.168.142.135` (LAN IP)
4. Traffic goes through WireGuard tunnel to NAS port 443
5. NPM receives request, checks access list: `10.253.0.2` matches WireGuard range ✅
6. NPM terminates TLS (wildcard cert), forwards to `sonarr:8989` on `jsatk-net`
7. Sonarr responds

---

## Docker Container Stack

All containers run on a custom bridge network called `jsatk-net` (subnet `172.18.0.0/16`). This allows containers to reference each other by name (e.g., `http://sonarr:8989`) rather than by IP.

Two containers are exceptions to the `jsatk-net` rule:
- `cloudflareddns` runs on the default `bridge` network — it only needs outbound internet access and doesn't communicate with other containers
- `pihole` runs on the default `bridge` network — required so it can bind to port 53 on the host and act as a network-wide DNS server

### Container Reference

| Container | Internal Port | LAN Port | Role |
|---|---|---|---|
| `bazarr` | 6767 | 6767 | Subtitle management |
| `kometa` | — | — | Automated Plex collection management |
| `beszel` | 8090 | 8090 | System monitoring dashboard |
| `beszel-agent` | — | — | Metrics agent for beszel |
| `cloudflareddns` | — | — | Dynamic DNS updater |
| `flaresolverr` | 8191 | 8191 | Cloudflare bypass for Prowlarr |
| `kopia` | 51515 | 51515 | Backup (appdata + /boot → Mac Studio via SFTP) |
| `maintainerr` | 6246 | 6246 | Automated library cleanup |
| `nginx-proxy-manager` | 80/81/443 | 80/81/443 | Reverse proxy + SSL termination |
| `notifiarr` | 5454 | 5454 | Discord notification hub |
| `plex` | 32400 | 32400 | Media server |
| `prowlarr` | 9696 | 9696 | Indexer manager |
| `qbittorrent` | 8080 | 8080 | Torrent client |
| `qui` | 7476 | 7476 | Alternative qBittorrent UI |
| `radarr` | 7878 | 7878 | Movie automation |
| `sabnzbd` | 8080 | 7080 | Usenet downloader |
| `seerr` | 5055 | 5055 | Request management |
| `sonarr` | 8989 | 8989 | TV show automation |
| `tautulli` | 8181 | 8181 | Plex analytics |
| `pihole` | 80/53 | 8155 (web) / 53 (DNS) | Network-wide ad blocking + DNS (bridge network) |

### Inter-Container Communication

Because everything is on `jsatk-net`, containers reference each other by name:

| From | To | Address |
|---|---|---|
| Sonarr / Radarr | SABnzbd | `http://sabnzbd:8080` |
| Sonarr / Radarr | qBittorrent | `http://qbittorrent:8080` |
| Sonarr / Radarr | Prowlarr | `http://prowlarr:9696` |
| Prowlarr | FlareSolverr | `http://flaresolverr:8191` |
| Bazarr | Sonarr | `http://sonarr:8989` |
| Bazarr | Radarr | `http://radarr:7878` |
| Tautulli | Plex | `http://plex:32400` |
| Notifiarr | Sonarr / Radarr / etc. | by container name |
| Seerr | Sonarr | `http://sonarr:8989` |
| Seerr | Radarr | `http://radarr:7878` |
| Seerr | Plex | `http://plex:32400` |

---

## Media Automation Pipeline

This is the core of the stack — the "arr" ecosystem that automates finding, downloading, and organising media.

### High-Level Flow

```
User request (Seerr)
    │
    ├─→ Sonarr (TV)  ─┐
    │                  ├─→ Prowlarr ──→ FlareSolverr (if needed)
    └─→ Radarr (Movies)┘       │
                               ├─→ Usenet indexers (NZB)
                               └─→ Torrent trackers
                                       │
                           ┌───────────┴───────────┐
                           ▼                       ▼
                       SABnzbd               qBittorrent
                     (Usenet DL)            (Torrent DL)
                           │                       │
                           └───────────┬───────────┘
                                       ▼
                              Hardlinked into media library
                                       │
                                       ▼
                                    Bazarr
                               (subtitles fetched)
                                       │
                                       ▼
                                 Plex library updated
```

### Seerr

Seerr is the user-facing request portal at `seerr.jsatk.us`. It's the only service exposed publicly (Cloudflare-proxied). Local sign-in is disabled — authentication is **Plex OAuth only**.

Users: `jsatk` (owner) and `lcatk`, both Plex-linked. When a request is approved, Seerr routes it to Sonarr (TV) or Radarr (movies).

### Prowlarr

Prowlarr is the centralised indexer manager. Rather than configuring indexers separately in Sonarr and Radarr, you configure them once in Prowlarr and it syncs to both automatically. Prowlarr manages both **Usenet indexers** (NZB) and **torrent trackers**.

### FlareSolverr

Some torrent tracker sites use Cloudflare's browser challenge to block scrapers. FlareSolverr runs a headless browser that can solve these challenges. Prowlarr is configured to route requests through FlareSolverr (`http://flaresolverr:8191`) for indexers that require it.

### Sonarr & Radarr

Sonarr handles TV shows; Radarr handles movies. Both:

1. Receive requests from Seerr
2. Search indexers via Prowlarr
3. Evaluate results against quality profiles
4. Send download jobs to SABnzbd (Usenet) or qBittorrent (torrents)
5. Monitor downloads and **hardlink** completed files into the media library
6. Rename files according to naming conventions
7. Notify Plex to refresh the library

### Download Clients

**SABnzbd (Usenet):**
- Primary client for Usenet (NZB) content
- Host port `7080` (internal container port `8080`)
- Newshosting is the primary Usenet provider (US-based)
- Eweka is configured as a backup provider for filling in incomplete articles
- `host_whitelist` includes: `sabnzbd`, `radarr`, `sonarr`, `sabnzbd.jsatk.us`, `jsatk-tower`

**qBittorrent (Torrents):**
- Secondary client for torrent content
- UI theme: VueTorrent (configured via `WebUI\RootFolder=/config/vuetorrent/`)
- **Qui** (running at `:7476`) provides an additional alternative qBittorrent frontend

### Hardlinks & File Layout

The *arr stack uses hardlinks so that files don't need to be duplicated between the download directory and the media library — the same underlying data is referenced from two paths without consuming extra disk space.

For hardlinks to work, the download directory and media library must be on the **same underlying filesystem**. This requires using disk paths (`/mnt/disk1/...`) rather than the FUSE overlay (`/mnt/user/`), since hardlinks cannot span the FUSE boundary.

Typical layout:

```
/mnt/user/data/
├── torrents/           ← qBittorrent downloads here
├── usenet/
│   ├── incomplete/     ← SABnzbd active downloads (in-progress)
│   └── complete/
│       ├── movies/     ← Radarr moves files from here into media/
│       └── tv/         ← Sonarr moves files from here into media/
└── media/
    ├── movies/         ← Radarr-managed library (hardlinked from torrents; moved from usenet)
    └── tv/             ← Sonarr-managed library (hardlinked from torrents; moved from usenet)
```

### Bazarr (Subtitles)

Bazarr monitors Sonarr and Radarr libraries and automatically fetches subtitles for new content.

Configured providers:
- **OpenSubtitles.com** (primary)
- **Addic7ed** (primary)

### Maintainerr (Library Cleanup)

Maintainerr automates removal of content from the Plex library based on configurable rules.

Example rule in use: **Jeopardy! episodes are automatically deleted 7 days after being watched.**

A notable configuration detail: there was a TVDB ID mismatch between Plex (ID `430963`) and Sonarr (ID `77075`) for Jeopardy!, which required explicit mapping in Maintainerr to get the rule to match correctly.

### Kometa (Plex Collection Management)

Kometa is a Docker container on `jsatk-net` that automatically creates and maintains Plex collections by querying external sources (Trakt, IMDb lists, etc.) and syncing them to the Plex library. It runs persistently and processes the library on a schedule — the default run time is **3:00 AM daily** via the `KOMETA_TIMES` environment variable.

Config is stored at `/mnt/user/appdata/kometa/config/config.yml`. Kometa connects to Plex at `http://plex:32400` on `jsatk-net`.

Kometa has no web UI and is not exposed via NPM. Logs can be viewed via the Unraid Docker log viewer.

### Tagarr (Release Group Tagging)

Tagarr is a shell script — not a standalone container — that tags imports in Radarr and Sonarr based on release group quality. It runs inside the Radarr and Sonarr containers: the script is stored at `/mnt/user/appdata/scripts/taggar/tagarr_import.sh` (note: directory is spelled `taggar`) and mounted into both containers at `/scripts/tagarr_import.sh`, invoked via a **Custom Script connection** in each app's notification settings.

Triggers configured in both Radarr and Sonarr:
- **On File Import** — tags newly imported files
- **On File Upgrade** — re-tags when a quality upgrade replaces an existing file
- **On Movie/Episode File Delete** — cleans up tags when files are removed

The script inspects the release group of each imported file and applies the appropriate Radarr/Sonarr tag, making it easy to identify which items in the library came from top-tier release groups at a glance.

---

## Plex & Playback

### Container Configuration

- **Image:** `ghcr.io/hotio/plex:latest`
- **Server name:** `jsatk-tower`
- **Machine ID:** `bdf765f130912067de6b632ab1c4351676ae464a`

Key Docker config:

| Config | Value | Purpose |
|---|---|---|
| `/dev/dri` passthrough | ✅ | Intel QuickSync HW transcoding |
| `/dev/shm/plex-transcode` → `/transcode` | tmpfs | RAM-based transcode temp files |
| `/mnt/user/data/media/` → `/data/media` | Volume | Media library |
| `PLEX_ADVERTISE_URL` | `http://jsatk-tower.local:32400,`<br>`http://192.168.142.135:32400,`<br>`https://plex.jsatk.us` | Multi-address advertising |
| `PLEX_PURGE_CODECS=true` | ✅ | Forces fresh codec download on startup |

### Hardware Transcoding

The N100 processor supports Intel QuickSync via VA-API. The hotio Plex image bundles the Intel iHD driver itself in `/config/Drivers/` — no system-level driver install is needed on Unraid. The image auto-detects `/dev/dri` on startup.

Historical fix notes:
- `/transcode` permission denied → fixed by mapping `/dev/shm/plex-transcode` (tmpfs in RAM) to `/transcode`
- EAE binary not executable → fixed by `PLEX_PURGE_CODECS=true`, which causes Plex to download fresh codecs with correct permissions on startup

### RAM Transcoding

Transcode temp files are written to `/dev/shm/plex-transcode`, a `tmpfs` mount living in RAM. This avoids writes to spinning disk during transcoding and improves performance for concurrent streams.

### Audio Passthrough

Plex is configured for audio passthrough to Apple TV, allowing lossless/surround formats (TrueHD, DTS-MA, etc.) to pass through without transcoding.

### Tautulli

Tautulli provides Plex analytics — watch history, user stats, stream quality breakdowns, etc. Connects to Plex at `http://plex:32400`.

---

## Monitoring & Notifications

### Notifiarr

Notifiarr is the notification hub. It receives events from Sonarr, Radarr, and other *arr apps and routes them to Discord webhooks. Notifiarr mounts `/mnt/disk1` at `/storage/1` to allow disk-level reporting.

Typical notifications sent to Discord:
- Download grabbed / completed
- New episode or movie added to library
- Quality upgrade replaced an existing file
- Grab or import failure

### Beszel + Beszel Agent

Beszel is a lightweight system monitoring dashboard at `:8090`. The `beszel-agent` container runs alongside it, collecting metrics (CPU, RAM, disk, network) from the host and feeding them to the Beszel UI. The agent mounts `/var/run/docker.sock` to report container-level stats as well.

---

## Backup

### Kopia

Kopia is a Docker container (`kopia/kopia:latest`) on `jsatk-net`, running at port `51515`, accessible at `kopia.jsatk.us` via NPM (LAN/VPN only). It handles encrypted, deduplicated offsite backup of appdata and the Unraid flash drive to the Mac Studio.

**Container config:**

| Setting | Value |
|---|---|
| Image | `kopia/kopia:latest` |
| Port | 51515 |
| Hostname | `jsatk-tower` |
| UI login | `jsatk` / separate UI password |
| URL | `https://kopia.jsatk.us` |

**Repository:**

| Setting | Value |
|---|---|
| Transport | SFTP to Mac Studio |
| SFTP target | `jsatk@192.168.142.152` |
| Repo path | `/Users/jsatk/Documents/unraid-backups/kopia` |
| SSH key (container) | `/root/.ssh/kopia_ed25519` |
| SSH mode | External password-less SSH command with `-i /root/.ssh/kopia_ed25519 -o StrictHostKeyChecking=no` |
| Encryption | `KOPIA_PASSWORD` env var (repo encryption password) |

The SSH key pair lives at `/boot/config/ssh/kopia_ed25519` (persists across reboots via the flash drive).

**Snapshot sources:**

| Source | Mount | Schedule | Notes |
|---|---|---|---|
| `/app/data` | → `/mnt/user/appdata` (rw) | Daily at 05:30 (via User Script) | Excludes plex cache/codecs/crash reports/logs, kometa/logs, nginx-proxy-manager/logs, kopia itself |
| `/boot` | — | Daily at 05:25 (Kopia scheduler) | Unraid flash drive |

**Retention policy:** 7 daily, 4 weekly, 3 monthly. Compression: `pgzip`.

**User Script (`kopia-backup`):**

Runs daily at `05:30 AM` (cron: `30 5 * * *`). Procedure:

1. Stops containers: `plex radarr sonarr prowlarr qbittorrent sabnzbd bazarr notifiarr tautulli seerr maintainerr` (with `--time=30`)
2. Sleeps 5 seconds
3. Runs `docker exec kopia kopia snapshot create /app/data`
4. Restarts all stopped containers

The `/boot` snapshot runs at 05:25 via Kopia's own built-in scheduler — no separate script needed for it.

**Scripts location:** All helper scripts consolidated under `/mnt/user/appdata/scripts/`.

---

## Storage Layout

### Volume Mapping Conventions

All app config is stored in `/mnt/cache/appdata/<app>/` — the cache drive is used for fast, frequently-written config data. Media and bulk data lives on the array at `/mnt/user/data/`.

| Path | Contents |
|---|---|
| `/mnt/cache/appdata/<app>/` | Container config, databases (bazarr, plex, prowlarr, qbittorrent, etc.) |
| `/mnt/user/appdata/<app>/` | Some app data (maintainerr, NPM, etc.) |
| `/mnt/user/data/media/` | Plex media library (movies, TV) |
| `/mnt/user/data/torrents/` | qBittorrent active downloads |
| `/mnt/user/data/usenet/` | SABnzbd active downloads (incomplete + complete) |

### Usenet Download Flow

The `data` share is configured with **Primary Cache: Yes, Secondary: Array**. This means SABnzbd writes downloads to the NVMe SSD cache first (fast writes), and Unraid's Mover runs hourly to transfer completed data to the array. The practical effect is:

- `data/usenet/incomplete/` — active downloads land on SSD during transfer
- `data/usenet/complete/movies` and `data/usenet/complete/tv` — completed downloads sit on SSD briefly, then get moved to array by Mover
- Radarr/Sonarr import from `complete/` and move files into `data/media/` (no hardlink needed — unlike torrents, there's no seeding to maintain)

This keeps download I/O fast without permanently consuming SSD space.

### Mover Tuning

By default, Unraid's mover cannot relocate files that are actively in use. This is a problem for any download client that holds open file handles on files while seeding — those files will sit on the NVMe cache indefinitely and never get moved to the array.

The **Mover Tuning** plugin (installed via Unraid Community Apps) solves this intelligently. Rather than running the mover blindly, it uses a pair of hook scripts to pause the relevant torrents in qBittorrent before the mover runs, then resume them once the mover finishes. This ensures files are only moved when they aren't being held open, preserving hardlinks and keeping the cache from gradually filling up.

Setup follows the [TRaSH Guides Mover Tuning guide](https://trash-guides.info/Downloaders/qBittorrent/Tips/How-to-run-the-unRaid-mover-for-qBittorrent/). Three files from that guide are stored in `/mnt/user/appdata/scripts/qbt-mover/`:

- `mover-tuning-start.sh` — runs before the mover starts; pauses torrents within the configured age range that are on the cache drive
- `mover-tuning-stop.sh` — runs after the mover finishes; resumes those torrents
- `mover-tuning.cfg` — config file for the above scripts (qBittorrent connection details, age thresholds, etc.)

The Mover Tuning plugin is configured to call these scripts on its schedule, so the whole process is fully automated.

### SSH Access

SSH uses key-based authentication. The `authorized_keys` file is persisted at `/boot/config/ssh/authorized_keys` on the Unraid flash drive so it survives reboots (Unraid's root filesystem is ephemeral on each boot).

### Dotfiles Persistence

Unraid boots from a USB flash drive into a RAM-based filesystem. Everything under `/root/` — including `.bash_profile`, `.vimrc`, `.vim/`, and any other home directory config — is **wiped on every reboot**. The flash drive (`/boot`) is FAT32, which does not support symlinks, so files cannot be symlinked from `/root/` into `/boot/`.

The solution is to store dotfiles on the flash drive and restore them via the `/boot/config/go` script, which Unraid executes at the end of every boot.

Files are stored at `/boot/config/home/` and restored with:

```bash
# Restore root home dotfiles
cp -af /boot/config/home/. /root/
```

The `-af` flags use archive mode (preserves permissions, timestamps, handles directories recursively) and force-overwrite. The trailing `.` on the source path copies the directory's contents including hidden dotfiles.

This entry lives in `/boot/config/go` before the `emhttp` line. After making changes to dotfiles on the server, copy the updated file back to `/boot/config/home/` to persist it across reboots:

```bash
cp ~/.vimrc /boot/config/home/.vimrc
# etc.
```

---

## Access Reference

### Local (LAN) or via WireGuard

| Service | URL |
|---|---|
| Unraid WebUI | `http://jsatk-tower.local:82` |
| NPM Admin | `http://jsatk-tower.local:81` |
| Plex | `https://plex.jsatk.us` or `http://jsatk-tower.local:32400` |
| Seerr | `https://seerr.jsatk.us` |
| Sonarr | `https://sonarr.jsatk.us` |
| Radarr | `https://radarr.jsatk.us` |
| Prowlarr | `https://prowlarr.jsatk.us` |
| SABnzbd | `https://sabnzbd.jsatk.us` |
| qBittorrent | `https://qbittorrent.jsatk.us` |
| Bazarr | `https://bazarr.jsatk.us` |
| Tautulli | `https://tautulli.jsatk.us` |
| Notifiarr | `https://notifiarr.jsatk.us` |
| Maintainerr | `https://maintainerr.jsatk.us` |
| Beszel | `https://beszel.jsatk.us` |
| Qui (alt qBittorrent UI) | `https://qui.jsatk.us` |
| Pi-hole | `https://pihole.jsatk.us` |
| Kopia | `https://kopia.jsatk.us` |

### Publicly Accessible (No VPN Needed)

| Service | URL |
|---|---|
| Seerr | `https://seerr.jsatk.us` |
| Plex | `https://plex.jsatk.us` |

### WireGuard

Connect to `vpn.jsatk.us:51820` using your peer config. Once connected, all `*.jsatk.us` URLs above resolve to `192.168.142.135` and are accessible as if on LAN.

---

## Decisions & Considerations

### WireGuard vs Tailscale

Tailscale was the first remote access solution used on this setup and it worked well — setup is minimal, MagicDNS is convenient, and it handles NAT traversal automatically. However, it was replaced with WireGuard for the following reasons:

**Third-party dependency.** Tailscale is fundamentally a managed service. Even though the client is open source, the control plane (the coordination server that authenticates devices, distributes keys, and manages the network) is run by Tailscale, Inc. If Tailscale goes down, changes its pricing, or makes a policy decision, remote access to the server breaks. For a self-hosted setup built on the principle of owning your own infrastructure, outsourcing the control plane is a meaningful philosophical compromise.

**Privacy.** Every device on a Tailscale network authenticates through Tailscale's servers, which means Tailscale has metadata about your devices, when they connect, and to what. WireGuard running natively on Unraid means the only parties involved are you and your hardware.

**Fewer moving parts.** WireGuard is built directly into Unraid — there is no extra container to maintain, no external authentication flow, and no separate app ecosystem to keep in sync. A WireGuard peer config is a small text file. The tunnel either works or it doesn't, and there is very little that can go wrong between those two states. Tailscale introduces an abstraction layer (MagicDNS, ACLs, the admin console) that, while user-friendly, is also additional surface area for things to break.

The tradeoff is that WireGuard requires manual peer management — adding a new device means generating keys and distributing a config file yourself, whereas Tailscale handles this automatically. For a small number of personal devices, this is a negligible cost.

---

### Nginx Proxy Manager vs SWAG

[SWAG](https://docs.linuxserver.io/general/swag/) (Secure Web Application Gateway) is a LinuxServer.io image that bundles nginx, Certbot (for Let's Encrypt cert management), and Fail2ban into a single container. It is the more traditional and widely-documented choice in the self-hosted/homelab community, particularly for *arr stack setups.

NPM (in the `mgutt` fork used here) is a GUI-driven reverse proxy that handles cert management and proxy host configuration through a web interface, without requiring manual editing of nginx config files.

**Why NPM was chosen:**

NPM's web UI makes adding and modifying proxy hosts fast and low-friction — no config files to write or restart cycles to manage. For a stack that was being built out incrementally (adding containers one at a time, adjusting access rules, iterating on the networking setup), this was a significant ergonomic advantage. The `mgutt` fork in particular adds Unraid-specific quality-of-life improvements that make it fit more naturally into the Unraid ecosystem.

SWAG's approach — editing nginx conf files directly — is more powerful and more transparent, but also more demanding. SWAG does ship with preset proxy conf files for popular self-hosted apps (including most of the *arr stack), which reduces the manual work, but the workflow is still more involved than clicking through a UI.

SWAG also bundles Fail2ban natively, which is an advantage over NPM. In the current setup, Fail2ban is on the to-do list but not yet configured.

**When SWAG would be worth considering:**

- If fine-grained nginx configuration becomes necessary (custom headers, advanced rate limiting, complex routing rules) that NPM's UI doesn't expose
- If Fail2ban integration becomes a priority and the NPM-based approach feels awkward
- If the setup grows to the point where infrastructure-as-code (managing nginx config in version control) is preferable to a GUI

For the current scale and use case, NPM is the right tool. SWAG is not a better choice — it is a different set of tradeoffs, with more power at the cost of more manual work.

---

### Future Considerations

**Fail2ban**

Fail2ban is an intrusion prevention tool that monitors log files for repeated failed authentication attempts and automatically bans offending IP addresses via firewall rules. It is currently on the backlog. The most relevant surface to protect is Seerr, which is the only publicly exposed service. Fail2ban can be run as a standalone Docker container (LinuxServer publishes one) and configured to watch NPM's access logs. This should be a relatively straightforward addition when the time comes.

**Migrating to SWAG**

If NPM's limitations become a pain point — particularly around advanced nginx config or native Fail2ban integration — SWAG is the natural migration path. The networking setup (Cloudflare DNS, wildcard cert via DNS-01 challenge, `jsatk-net` bridge) translates directly; it would primarily be a matter of recreating proxy host configs as nginx `.conf` files and enabling the relevant preset confs from SWAG's bundled library. The migration is non-trivial but well-documented, and the existing setup would not require significant rearchitecting.
