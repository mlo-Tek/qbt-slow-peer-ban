# qbt-slowban for hotio/qbittorrent (Docker / Docker Compose)

> [!WARNING]
> ⚠️ **AI-assisted project**
>
> This project, including portions of the Python implementation, Docker setup, and documentation, was created with substantial assistance from **OpenAI ChatGPT** and subsequently reviewed and adapted for the intended setup.
>
> AI-generated or AI-assisted code can contain defects. Review the code and test it in your own environment before relying on it.

A lightweight Python sidecar for **hotio/qbittorrent**, run with Docker or Docker Compose.

This repository is a fork of [`TechClusterHQ/qbt-slowban`](https://github.com/TechClusterHQ/qbt-slowban), adapted from the original LinuxServer.io Docker Mod approach to run as a standalone sidecar container for hotio/qbittorrent.

## Features

- Designed as a separate sidecar container for hotio/qbittorrent
- Uses the qBittorrent Web API
- Tracks slow peers per torrent
- Also scans seeding torrents with active upload (leechers downloading slowly from you), not only downloading torrents
- Warning before a ban is applied
- Persistent state across container restarts
- Scheduled clearing of the qBittorrent manual ban list
- Optional permanent bans that survive scheduled clears
- Dry-run mode
- 2-hour rotating log files
- Configurable log retention
- Periodic status summaries
- Colored console output

## Default settings

| Setting | Default |
|---|---:|
| Warning after | 45 seconds |
| Ban after | 90 seconds |
| Minimum upload speed | 50,768 B/s |
| Poll interval | 10 seconds |
| Summary interval | 600 seconds |
| Clear manual bans | Every 12 hours |
| Log retention | 7 days |
| Dry run | false |

The minimum upload-speed value is expressed in **bytes per second**.

## Repository layout

```text
qbt-slowban-hotio/
├── slowban.py
├── runner.py
├── Dockerfile
├── docker-compose.yml
├── .env.example
├── README.md
├── SECURITY.md
└── .gitignore
```

## Installation

The image is published to GHCR as `ghcr.io/mlo-tek/qbt-slowban-hotio:latest`.

### Docker Compose (recommended)

1. Download the compose file and the example environment file:

   ```bash
   mkdir qbt-slowban && cd qbt-slowban
   curl -LO https://raw.githubusercontent.com/mlo-Tek/qbt-slowban-hotio/main/docker-compose.yml
   curl -L https://raw.githubusercontent.com/mlo-Tek/qbt-slowban-hotio/main/.env.example -o .env
   ```

2. Edit `.env` and set at minimum `QBT_URL`, `QBT_USERNAME` and `QBT_PASSWORD`.

3. Start it and inspect the log:

   ```bash
   docker compose up -d
   docker compose logs -f
   ```

State and logs are stored in `./state` and `./logs`. If qBittorrent runs in another compose project, attach the service to the same Docker network and use the qBittorrent container name in `QBT_URL`.

Update to the latest image:

```bash
docker compose pull && docker compose up -d
```

### Docker run

```bash
docker run -d --name qbt-slowban --restart unless-stopped \
  -e QBT_URL=http://192.168.1.100:8080 \
  -e QBT_USERNAME=admin -e QBT_PASSWORD=changeme \
  -e TZ=Europe/Berlin \
  -v "$PWD/state:/state" -v "$PWD/logs:/logs" \
  ghcr.io/mlo-tek/qbt-slowban-hotio:latest
```

### Build locally

```bash
docker build -t qbt-slowban .
```

## Important configuration variables

### qBittorrent

- `QBT_URL` — qBittorrent WebUI/API URL
- `QBT_USERNAME` — qBittorrent username
- `QBT_PASSWORD` — qBittorrent password

### Slow-peer detection

- `SLOWBAN_MIN_SPEED` — minimum upload speed in bytes per second
- `SLOWBAN_WARN_TIME` — seconds below the threshold before a warning is logged
- `SLOWBAN_THRESHOLD_TIME` — seconds below the threshold before the peer is banned
- `SLOWBAN_POLL_INTERVAL` — polling interval in seconds

`SLOWBAN_WARN_TIME` must be lower than `SLOWBAN_THRESHOLD_TIME`.

### Scheduled unban

`SLOWBAN_CLEAR_PERIODICALLY` accepts a 5-field cron expression.

Default:

```text
0 */12 * * *
```

This runs at 00:00 and 12:00 according to the configured container timezone.

`SLOWBAN_BANNED_PEERS` can contain comma-separated peers that should be kept permanently banned when the scheduled clear runs.

### Logging

- `SLOWBAN_LOG_LEVEL`
- `SLOWBAN_LOG_DIR`
- `SLOWBAN_LOG_RETENTION_DAYS`
- `SLOWBAN_LOG_UNBAN_DETAILS`
- `SLOWBAN_COLOR_LOGS`
- `SLOWBAN_SUMMARY_INTERVAL`

Log files are split into 2-hour time slots.

## Security

Do **not** commit a populated `.env` or compose file containing your real qBittorrent username, password, internal IP addresses, or other private configuration.

`.env.example` intentionally contains only generic example values and `.env` is git-ignored.

See [`SECURITY.md`](SECURITY.md) for additional notes.

## Disclaimer

Use at your own risk. Banning peers and manipulating the qBittorrent manual ban list can affect active transfers and connectivity. Test with `SLOWBAN_DRY_RUN=true` first if you want to verify behavior without applying real bans.
