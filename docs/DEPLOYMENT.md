# Deployment & Operations

## Hosting

**Recommendation: Hetzner CAX11 ARM VPS** — EUR 3.79/mo (~$5), 2 vCPU, 4GB RAM, 40GB disk.

The workload is bursty outbound HTTP (LLM calls, twitterapi.io). Discord Gateway WebSockets idle 99% of the time. Actual need: 1 vCPU, 512MB-1GB RAM, 10GB disk. Hetzner is overkill and still under $5/mo.

Alternatives: Fly.io, Railway — 2-3x more expensive, add platform-specific failure modes (volume migrations, ephemeral storage). A plain VPS with pm2 or systemd is simpler.

## Monthly Cost Estimate

| Component | Cost/mo |
|-----------|---------|
| Hetzner VPS | ~$5 |
| Domain | ~$1 |
| twitterapi.io | ~$9 |
| Anthropic API (Haiku + Sonnet) | ~$10-20 |
| **Total** | **~$25-35/mo** |

## Process Management

### Option A: pm2

```bash
# Install pm2 globally
npm install -g pm2

# Start Podders
pm2 start dist/index.js --name podders -- run

# Auto-restart on crash, save process list for reboot persistence
pm2 save
pm2 startup  # generates systemd/init script for auto-start on boot
```

`ecosystem.config.cjs` (optional, for pm2 config-as-code):

```js
module.exports = {
  apps: [{
    name: 'podders',
    script: 'dist/index.js',
    args: 'run',
    env_file: '.env',
    max_memory_restart: '512M',
    restart_delay: 5000,
  }],
};
```

### Option B: systemd

```ini
# /etc/systemd/system/podders.service
[Unit]
Description=Podders v2 — market intelligence engine
After=network.target

[Service]
Type=simple
User=podders
WorkingDirectory=/opt/podders
EnvironmentFile=/opt/podders/.env
ExecStart=/usr/bin/node dist/index.js run
Restart=always
RestartSec=5
KillSignal=SIGTERM
TimeoutStopSec=30

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable podders
sudo systemctl start podders
```

### Health Check

```bash
# Cron-based health check (add to crontab)
* * * * * curl -sf http://localhost:3000/api/v1/status || systemctl restart podders
```

## TLS / HTTPS

**Caddy** as reverse proxy — automatic HTTPS with Let's Encrypt, ~10MB RAM.

```
# /etc/caddy/Caddyfile
podders.yourdomain.com {
    reverse_proxy localhost:3000
}
```

Alternative: **Cloudflare Tunnel** (free) if you don't want to open ports 80/443.

## Monitoring

### Logs

Pino JSON logs go to stdout. Options:
- **Minimum (pm2)**: `pm2 logs podders` — pm2 manages log files in `~/.pm2/logs/`, add `pm2-logrotate` module
- **Minimum (systemd)**: `journalctl -u podders -f` — systemd journal handles rotation
- **Better**: Better Stack (Logtail) free tier (1GB/mo) — pipe stdout via `pino-transport`
- **Full**: Loki + Grafana on same VPS (~200MB RAM overhead)

### Uptime

External health check on `/api/v1/status` every 60s. Options:
- **Uptime Kuma** self-hosted on same VPS (10MB RAM)
- **Better Stack** free tier (10 monitors)
- Alert to Discord webhook — already have delivery infrastructure

## Deploys

### Procedure

```bash
# Pull latest code
cd /opt/podders && git pull

# Install deps + build
npm install && npm run build

# Restart (pm2)
pm2 restart podders

# Or restart (systemd)
sudo systemctl restart podders
```

### Message Loss During Restart

Discord Gateway disconnects during restart (~5-10s). Messages received by Discord while disconnected are **lost** — user tokens don't get replay. At 5K items/day (~3/min), expect 0-2 lost messages per deploy. Acceptable.

Crash recovery handles in-flight LLM work: orphaned `processing` items reset to `ready` on startup.

## Backup

### Daily Backup

`pg_dump` after entity decay cron (DB quietest at 00:10 UTC). Timestamped files in `backups/`, retain 7 days. Uses `--format=custom` for compression and selective restore.

```bash
pg_dump --format=custom --file=backups/podders-$(date +%Y%m%d-%H%M%S).dump "$DATABASE_URL"
```

### Verification

After backup, verify the dump is valid and contains data:

```bash
pg_restore --list backups/podders-TIMESTAMP.dump > /dev/null 2>&1
if [ $? -ne 0 ]; then
  echo "Backup verification failed"
  exit 1
fi
```

### Restore

```bash
pm2 stop podders  # or: sudo systemctl stop podders
pg_restore --clean --if-exists --dbname="$DATABASE_URL" backups/podders-TIMESTAMP.dump
pm2 start podders  # or: sudo systemctl start podders
```

## Secret Rotation

| Secret | Procedure | Downtime |
|--------|-----------|----------|
| `API_KEY` | `POST /api/v1/auth/rotate` | None (live) |
| `DISCORD_TOKENS` | Update `.env`, restart | ~5-10s |
| `ANTHROPIC_API_KEY` | Update `.env`, restart | ~5-10s |
| `GEMINI_API_KEY` | Update `.env`, restart | ~5-10s |
| `TWITTERAPI_KEY` | Update `.env`, restart | ~5-10s |

## Postgres Storage

With 30-day item retention, DB plateaus after ~30 days:

| Table | Rows (30-day) | Size |
|-------|--------------|------|
| items | ~150K | ~150MB |
| summaries (90d) | ~1,000 | ~10MB |
| entities | ~2-5K | ~5MB |
| reports | grows forever | ~1MB/year |
| indexes | — | ~30MB |
| **Total** | | **~200MB stable** |

### Maintenance

- **Weekly VACUUM ANALYZE**: reclaim space and update planner statistics after bulk retention deletes
- **REINDEX CONCURRENTLY** if index bloat exceeds 30% (rare with retention)
- DB size is stable — retention caps growth
