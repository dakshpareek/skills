---
name: qbittorrent
description: "Manage qBittorrent via its Web API: list, add, pause, resume, delete torrents, check transfer speeds, and configure preferences. Use when the user asks about torrents, downloads, seeding, or qBittorrent management."
compatibility: Requires curl and jq. Requires qBittorrent with Web UI enabled.
---

# qBittorrent Management

Interact with qBittorrent through its Web API using `curl`.

## Configuration

The user must have qBittorrent running with Web UI enabled. Default connection:

- **Host**: `localhost`
- **Port**: `8080`
- **Username**: `admin`
- **Password**: check qBittorrent preferences (default on first run is shown in the log)

Ask the user for their host, port, and credentials if not known. Store the session cookie for subsequent requests.

## Authentication

All API calls require authentication. Log in first to get a session cookie:

```bash
# Log in and save the session cookie
curl -s -c /tmp/qbt-cookie.txt \
  --data "username=admin&password=PASSWORD" \
  "http://localhost:8080/api/v2/auth/login"
# Response: "Ok." on success, "Fails." on failure
```

Use `-b /tmp/qbt-cookie.txt` on all subsequent requests.

## Common Operations

### List all torrents

```bash
curl -s -b /tmp/qbt-cookie.txt \
  "http://localhost:8080/api/v2/torrents/info" | jq '.'
```

Filter by status: `?filter=downloading`, `?filter=seeding`, `?filter=completed`, `?filter=paused`, `?filter=active`, `?filter=inactive`, `?filter=errored`, `?filter=stopped`, `?filter=running`

Filter by category: `?category=mycategory`

Sort results: `?sort=name`, `?sort=size`, `?sort=progress`, `?sort=dlspeed`, `?sort=upspeed`, `?sort=ratio`, `?sort=added_on`

Reverse order: `?reverse=true`

Limit/offset: `?limit=10&offset=0`

### Show a concise torrent summary

Use `jq` to format a readable table:

```bash
curl -s -b /tmp/qbt-cookie.txt \
  "http://localhost:8080/api/v2/torrents/info" | \
  jq -r '.[] | "\(.name)\t\(.state)\t\((.progress * 100 | floor))%\t\((.dlspeed / 1048576 * 100 | floor / 100))MB/s\t\((.size / 1073741824 * 100 | floor / 100))GB"'
```

### Add a torrent by URL or magnet link

```bash
curl -s -b /tmp/qbt-cookie.txt \
  --data-urlencode "urls=magnet:?xt=urn:btih:HASH..." \
  "http://localhost:8080/api/v2/torrents/add"
```

Optional parameters (append with `--data-urlencode`):
- `savepath=/path/to/save` — save location
- `category=mycategory` — assign category
- `tags=tag1,tag2` — assign tags
- `paused=true` — add in paused/stopped state
- `dlLimit=1048576` — download speed limit in bytes/s
- `upLimit=1048576` — upload speed limit in bytes/s
- `sequentialDownload=true` — enable sequential download
- `firstLastPiecePrio=true` — prioritize first/last pieces

### Add a torrent from a .torrent file

```bash
curl -s -b /tmp/qbt-cookie.txt \
  -F "torrents=@/path/to/file.torrent" \
  "http://localhost:8080/api/v2/torrents/add"
```

### Pause (stop) torrents

```bash
# Pause specific torrents (pipe-separated hashes)
curl -s -b /tmp/qbt-cookie.txt \
  --data "hashes=HASH1|HASH2" \
  "http://localhost:8080/api/v2/torrents/stop"

# Pause all
curl -s -b /tmp/qbt-cookie.txt \
  --data "hashes=all" \
  "http://localhost:8080/api/v2/torrents/stop"
```

### Resume (start) torrents

```bash
curl -s -b /tmp/qbt-cookie.txt \
  --data "hashes=HASH1|HASH2" \
  "http://localhost:8080/api/v2/torrents/start"

# Resume all
curl -s -b /tmp/qbt-cookie.txt \
  --data "hashes=all" \
  "http://localhost:8080/api/v2/torrents/start"
```

### Delete torrents

```bash
# Delete torrent but keep files
curl -s -b /tmp/qbt-cookie.txt \
  --data "hashes=HASH1|HASH2&deleteFiles=false" \
  "http://localhost:8080/api/v2/torrents/delete"

# Delete torrent AND files
curl -s -b /tmp/qbt-cookie.txt \
  --data "hashes=HASH1|HASH2&deleteFiles=true" \
  "http://localhost:8080/api/v2/torrents/delete"
```

**Always confirm with the user before using `deleteFiles=true`.**

### Get torrent details

```bash
curl -s -b /tmp/qbt-cookie.txt \
  "http://localhost:8080/api/v2/torrents/properties?hash=TORRENT_HASH" | jq '.'
```

### Get torrent files

```bash
curl -s -b /tmp/qbt-cookie.txt \
  "http://localhost:8080/api/v2/torrents/files?hash=TORRENT_HASH" | jq '.'
```

### Get torrent trackers

```bash
curl -s -b /tmp/qbt-cookie.txt \
  "http://localhost:8080/api/v2/torrents/trackers?hash=TORRENT_HASH" | jq '.'
```

### Set torrent speed limits

```bash
# Download limit (bytes/s), 0 = unlimited
curl -s -b /tmp/qbt-cookie.txt \
  --data "hashes=HASH&limit=1048576" \
  "http://localhost:8080/api/v2/torrents/setDownloadLimit"

# Upload limit (bytes/s), 0 = unlimited
curl -s -b /tmp/qbt-cookie.txt \
  --data "hashes=HASH&limit=524288" \
  "http://localhost:8080/api/v2/torrents/setUploadLimit"
```

### Set torrent location

```bash
curl -s -b /tmp/qbt-cookie.txt \
  --data "hashes=HASH&location=/new/path" \
  "http://localhost:8080/api/v2/torrents/setLocation"
```

### Set torrent category

```bash
curl -s -b /tmp/qbt-cookie.txt \
  --data "hashes=HASH&category=movies" \
  "http://localhost:8080/api/v2/torrents/setCategory"
```

### Add/remove tags

```bash
# Add tags
curl -s -b /tmp/qbt-cookie.txt \
  --data "hashes=HASH&tags=tag1,tag2" \
  "http://localhost:8080/api/v2/torrents/addTags"

# Remove tags
curl -s -b /tmp/qbt-cookie.txt \
  --data "hashes=HASH&tags=tag1" \
  "http://localhost:8080/api/v2/torrents/removeTags"
```

## Transfer Info

### Global transfer speed

```bash
curl -s -b /tmp/qbt-cookie.txt \
  "http://localhost:8080/api/v2/transfer/info" | jq '.'
```

Key fields: `dl_info_speed` (bytes/s), `up_info_speed` (bytes/s), `dl_info_data` (total downloaded), `up_info_data` (total uploaded).

### Toggle speed limits mode (alt speed)

```bash
curl -s -b /tmp/qbt-cookie.txt \
  "http://localhost:8080/api/v2/transfer/toggleSpeedLimitsMode"
```

### Set global download/upload limits

```bash
# Global download limit (bytes/s), 0 = unlimited
curl -s -b /tmp/qbt-cookie.txt \
  --data "limit=5242880" \
  "http://localhost:8080/api/v2/transfer/setDownloadLimit"

# Global upload limit
curl -s -b /tmp/qbt-cookie.txt \
  --data "limit=1048576" \
  "http://localhost:8080/api/v2/transfer/setUploadLimit"
```

## Categories

```bash
# List categories
curl -s -b /tmp/qbt-cookie.txt \
  "http://localhost:8080/api/v2/torrents/categories" | jq '.'

# Create category
curl -s -b /tmp/qbt-cookie.txt \
  --data-urlencode "category=movies" \
  --data-urlencode "savePath=/downloads/movies" \
  "http://localhost:8080/api/v2/torrents/createCategory"

# Remove categories
curl -s -b /tmp/qbt-cookie.txt \
  --data "categories=movies" \
  "http://localhost:8080/api/v2/torrents/removeCategories"
```

## Application

```bash
# Get qBittorrent version
curl -s -b /tmp/qbt-cookie.txt "http://localhost:8080/api/v2/app/version"

# Get API version
curl -s -b /tmp/qbt-cookie.txt "http://localhost:8080/api/v2/app/webapiVersion"

# Get preferences
curl -s -b /tmp/qbt-cookie.txt "http://localhost:8080/api/v2/app/preferences" | jq '.'

# Set preferences (JSON body)
curl -s -b /tmp/qbt-cookie.txt \
  --data-urlencode 'json={"download_path":"/downloads","max_connec":500}' \
  "http://localhost:8080/api/v2/app/setPreferences"

# Get default save path
curl -s -b /tmp/qbt-cookie.txt "http://localhost:8080/api/v2/app/defaultSavePath"
```

## Search

```bash
# Start search
curl -s -b /tmp/qbt-cookie.txt \
  --data "pattern=search+query&plugins=all&category=all" \
  "http://localhost:8080/api/v2/search/start" | jq '.id'

# Get search results (use id from start response)
curl -s -b /tmp/qbt-cookie.txt \
  "http://localhost:8080/api/v2/search/results?id=SEARCH_ID&limit=20" | jq '.'

# Stop search
curl -s -b /tmp/qbt-cookie.txt \
  --data "id=SEARCH_ID" \
  "http://localhost:8080/api/v2/search/stop"

# Delete search
curl -s -b /tmp/qbt-cookie.txt \
  --data "id=SEARCH_ID" \
  "http://localhost:8080/api/v2/search/delete"

# List search plugins
curl -s -b /tmp/qbt-cookie.txt \
  "http://localhost:8080/api/v2/search/plugins" | jq '.'
```

## RSS

```bash
# List RSS items
curl -s -b /tmp/qbt-cookie.txt \
  "http://localhost:8080/api/v2/rss/items?withData=true" | jq '.'

# Add RSS feed
curl -s -b /tmp/qbt-cookie.txt \
  --data-urlencode "url=https://example.com/rss" \
  --data-urlencode "path=MyFeed" \
  "http://localhost:8080/api/v2/rss/addFeed"

# Remove RSS item
curl -s -b /tmp/qbt-cookie.txt \
  --data-urlencode "path=MyFeed" \
  "http://localhost:8080/api/v2/rss/removeItem"
```

## Important Notes

- **Byte conversion**: speeds are in bytes/s. 1 MB/s = 1048576 bytes/s.
- **Hashes**: use pipe `|` to separate multiple torrent hashes, or `all` for all torrents.
- **Cookie expiry**: if you get 403 errors, re-authenticate.
- **qBittorrent v5.0+**: `pause`/`resume` endpoints renamed to `stop`/`start`. Use `stop`/`start` for v5.0+.
- **Never log or expose the user's password** in output.

## Helper Script

Run `scripts/qbt.sh` for a convenient wrapper around common operations:

```bash
scripts/qbt.sh login <host> <port> <user> <pass>
scripts/qbt.sh list [filter]
scripts/qbt.sh add <url_or_magnet>
scripts/qbt.sh pause <hash|all>
scripts/qbt.sh resume <hash|all>
scripts/qbt.sh delete <hash> [--with-files]
scripts/qbt.sh info <hash>
scripts/qbt.sh speed
```
