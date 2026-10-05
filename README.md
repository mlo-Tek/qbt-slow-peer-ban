# qbt-slow-peer-ban for qBittorrent (Docker, Docker Compose & Unraid)

> [!WARNING]
> ⚠️ **AI-assisted project**
>
> This project, including portions of the Python implementation, Docker/Unraid setup, and documentation, was created with substantial assistance from **OpenAI ChatGPT** and subsequently reviewed and adapted for the intended setup.
>
> AI-generated or AI-assisted code can contain defects. Review the code and test it in your own environment before relying on it.

A lightweight Python helper container for **qBittorrent** (developed with hotio/qbittorrent and reported to work with binhex/arch-qbittorrentvpn; any qBittorrent container with a reachable WebUI should work), run with Docker, Docker Compose or Unraid.

This is an independent helper container implementation. The idea of banning slow peers was inspired by [`TechClusterHQ/qbt-slowban`](https://github.com/TechClusterHQ/qbt-slowban), a LinuxServer.io Docker Mod; this project does not depend on it and runs as a standalone container.

## Features

- Separate helper container for qBittorrent (hotio, binhex, ...), runs with Docker, Docker Compose and Unraid
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

All settings are environment variables. Only `QBT_URL`, `QBT_USERNAME` and `QBT_PASSWORD` normally need to be set. Everything else is optional. Details and examples are in [Configuration reference](#configuration-reference).

| Variable | Built-in default | Meaning |
|---|---:|---|
| `QBT_URL` | `http://10.20.20.15:8080` | qBittorrent WebUI URL (always set this) |
| `QBT_USERNAME` / `QBT_PASSWORD` | empty | qBittorrent login |
| `SLOWBAN_MIN_SPEED` | `100000` B/s (100 kB/s) | Peers downloading slower than this (but above 0) count as slow |
| `SLOWBAN_WARN_TIME` | `90` s | Warning after this long below the minimum speed |
| `SLOWBAN_THRESHOLD_TIME` | `180` s | Ban after this long below the minimum speed |
| `SLOWBAN_POLL_INTERVAL` | `10` s | How often qBittorrent is checked |
| `SLOWBAN_SUMMARY_INTERVAL` | `600` s | How often a status summary is logged |
| `SLOWBAN_CLEAR_PERIODICALLY` | empty (off) | Cron schedule to clear the manual ban list |
| `SLOWBAN_BANNED_PEERS` | empty | Peers kept banned when the list is cleared |
| `SLOWBAN_LOG_LEVEL` | `INFO` | Minimum level that is logged |
| `SLOWBAN_LOG_DIR` | `/logs` | Directory for the log files |
| `SLOWBAN_LOG_RETENTION_DAYS` | `7` | Log files older than this are deleted |
| `SLOWBAN_LOG_UNBAN_DETAILS` | `false` | Log every peer that is unbanned |
| `SLOWBAN_COLOR_LOGS` | `true` | Colored console output |
| `SLOWBAN_DRY_RUN` | `false` | Only log what would happen, never ban |
| `SLOWBAN_STATE_FILE` | `/state/slowban_state.json` | Persistent state file |

The Unraid template ships with warning after **45 s**, ban after **90 s** and the periodic unban set to `0 */12 * * *` (every 12 hours). The speed value is in **bytes per second**, while qBittorrent's WebUI shows kB/s: multiply the kB/s value by 1000 (`100000` = 100 kB/s, `50000` = 50 kB/s).

## Repository layout

```text
qbt-slow-peer-ban/
├── slowban.py
├── runner.py
├── Dockerfile
├── docker-compose.yml
├── .env.example
├── qbt-slow-peer-ban.xml      (Unraid template)
├── assets/                 (template icon)
├── ca_profile.xml          (Unraid Community Applications profile)
├── LICENSE
├── README.md
├── SECURITY.md
└── .gitignore
```

## Installation

The image is published to GHCR as `ghcr.io/mlo-tek/qbt-slow-peer-ban:latest`.

### Docker Compose (recommended)

1. Download the compose file and the example environment file:

   ```bash
   mkdir qbt-slow-peer-ban && cd qbt-slow-peer-ban
   curl -LO https://raw.githubusercontent.com/mlo-Tek/qbt-slow-peer-ban/main/docker-compose.yml
   curl -L https://raw.githubusercontent.com/mlo-Tek/qbt-slow-peer-ban/main/.env.example -o .env
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
docker run -d --name qbt-slow-peer-ban --restart unless-stopped \
  -e QBT_URL=http://192.168.1.100:8080 \
  -e QBT_USERNAME=admin -e QBT_PASSWORD=changeme \
  -e TZ=Europe/Berlin \
  -v "$PWD/state:/state" -v "$PWD/logs:/logs" \
  ghcr.io/mlo-tek/qbt-slow-peer-ban:latest
```

### Unraid

Create the appdata directories:

```bash
mkdir -p /mnt/user/appdata/qbt-slow-peer-ban/{state,logs}
```

Download the Unraid template into the user-template directory:

```bash
curl -L https://raw.githubusercontent.com/mlo-Tek/qbt-slow-peer-ban/main/qbt-slow-peer-ban.xml \
  -o /boot/config/plugins/dockerMan/templates-user/qbt-slow-peer-ban.xml
```

Then open the Unraid Docker page, choose **Add Container** and select the `qbt-slow-peer-ban` template. The image is pulled from GHCR, so no further files are needed.

Set at minimum `QBT_URL`, `QBT_USERNAME` and `QBT_PASSWORD`. The template uses the `bridge` network by default; adjust it if your qBittorrent is on a custom network. Start the container and inspect its log.

Alternatively, Unraid users can run the Docker Compose file above via the Compose Manager plugin.

### Build locally

```bash
docker build -t qbt-slow-peer-ban .
```

## Configuration reference

### qBittorrent

- `QBT_URL` — address of the qBittorrent WebUI/API, e.g. `http://192.168.1.100:8080`.
- `QBT_USERNAME`, `QBT_PASSWORD` — WebUI login.

### Slow-peer detection

A peer is *slow* when it downloads from you at more than 0 B/s but less than `SLOWBAN_MIN_SPEED`. Peers that receive nothing are ignored.

- `SLOWBAN_MIN_SPEED` — speed limit in bytes per second. `100000` means about 100 KB/s.
- `SLOWBAN_WARN_TIME` — seconds a peer must stay below the limit before a warning is logged.
- `SLOWBAN_THRESHOLD_TIME` — seconds below the limit before the peer is banned. Must be higher than `SLOWBAN_WARN_TIME`.
- `SLOWBAN_POLL_INTERVAL` — how often (seconds) qBittorrent is queried.

Example: with `SLOWBAN_MIN_SPEED=100000`, `SLOWBAN_WARN_TIME=60` and `SLOWBAN_THRESHOLD_TIME=120`, a peer that stays below 100 KB/s gets a warning after 60 s and is banned after 120 s. If it speeds up above the limit (or stops downloading completely) in between, its timer is reset.

### Scheduled unban

- `SLOWBAN_CLEAR_PERIODICALLY` — 5-field cron expression (`minute hour day month weekday`) that clears the manual ban list so peers get a second chance. Empty means bans are never cleared automatically. Times use the container timezone (`TZ`).

  | Value | Runs |
  |---|---|
  | `0 */12 * * *` | at 00:00 and 12:00 |
  | `0 4 * * *` | daily at 04:00 |
  | `0 4 * * 0` | Sundays at 04:00 |

- `SLOWBAN_BANNED_PEERS` — comma-separated list of peers that stay banned permanently. They are re-applied every time the list is cleared. Example: `SLOWBAN_BANNED_PEERS=203.0.113.5,198.51.100.7`.

### Logging

- `SLOWBAN_LOG_LEVEL` — minimum level that is written. Available levels, from most to least detailed: `DEBUG`, `INFO`, `WARN`, `BAN`, `UNBAN`, `ERROR`. Example: `WARN` hides routine `INFO` lines (startup, summaries) but still shows warnings, bans, unbans and errors.
- `SLOWBAN_LOG_DIR` — directory inside the container where log files are written (default `/logs`). Map it to a host folder to keep the logs, e.g. `./logs:/logs`. Files are named like `slowban-2026-10-05_1400.log`, one per 2-hour time slot.
- `SLOWBAN_LOG_RETENTION_DAYS` — log files older than this many days are deleted automatically. Example: `7` keeps one week of logs.
- `SLOWBAN_LOG_UNBAN_DETAILS` — `true` logs every single peer that is unbanned during a scheduled clear. With `false` only a short summary line (how many were unbanned) is logged.
- `SLOWBAN_COLOR_LOGS` — `true` colors the console output (warnings yellow, bans red, and so on). Set `false` if your log viewer shows raw color codes. Log files are never colored.
- `SLOWBAN_SUMMARY_INTERVAL` — seconds between status summary lines. A summary shows the number of torrents, active and tracked slow peers, warnings, and total bans and unbans. Example: `600` logs one summary every 10 minutes.

### Other

- `SLOWBAN_DRY_RUN` — `true` only logs what would be banned or unbanned and changes nothing in qBittorrent. Good for testing new limits.
- `SLOWBAN_STATE_FILE` — where the tracking state is stored (default `/state/slowban_state.json`). Map `/state` to a host folder so it survives restarts.

## Security

Do **not** commit a populated `.env`, compose file or Unraid XML template containing your real qBittorrent username, password, internal IP addresses, or other private configuration.

`.env.example` and the Unraid template intentionally contain only generic example values; `.env` is git-ignored.

See [`SECURITY.md`](SECURITY.md) for additional notes.

## License

MIT, see [`LICENSE`](LICENSE).

## Disclaimer

Use at your own risk. Banning peers and manipulating the qBittorrent manual ban list can affect active transfers and connectivity. Test with `SLOWBAN_DRY_RUN=true` first if you want to verify behavior without applying real bans.
