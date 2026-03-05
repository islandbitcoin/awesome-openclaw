# OpenClaw Multi-Agent Dashboard

A lightweight, self-hosted monitoring dashboard for OpenClaw multi-agent setups. Tracks node health, agent activity, LLM costs, and infrastructure spend — all from a single HTML page backed by a tiny Node.js server.

**Live demo concept:** `https://monitor.yourdomain.com`

![Dashboard](https://img.shields.io/badge/stack-Node.js%20%2B%20HTML%20%2B%20Cloudflare%20Tunnel-blue)

---

## What It Does

- **Node monitoring** — CPU, RAM, disk usage for local Macs, remote VPS, and paired OpenClaw nodes
- **Agent activity** — tracks which agents are active, their last message, session count, and tool calls
- **Cost tracking** — parses OpenClaw session logs to estimate per-agent LLM spend (daily/weekly/monthly)
- **Subscription overview** — fixed monthly costs (API subscriptions, hosting, electricity)
- **Auto-refresh** — polls `/api/metrics` every 60s for live updates
- **Zero dependencies** — no React, no build step, no npm install. Pure Node.js + vanilla HTML/JS

---

## Architecture

```
┌─────────────────────────────────────────────────┐
│  Browser (dashboard.html)                        │
│  ← polls /api/metrics every 60s                  │
└──────────────┬──────────────────────────────────┘
               │ HTTP
┌──────────────▼──────────────────────────────────┐
│  server.mjs (port 8877, localhost only)          │
│  ├── /              → serves dashboard.html      │
│  └── /api/metrics   → aggregates all data        │
│      ├── macOS local metrics (top, vm_stat, df)  │
│      ├── SSH metrics from Mac nodes              │
│      ├── HTTP metrics from VPS nodes (:9100)     │
│      ├── parse-activity.mjs (session log parser) │
│      └── parse-costs.mjs (LLM cost estimator)   │
└──────────────┬──────────────────────────────────┘
               │ Cloudflare Tunnel
┌──────────────▼──────────────────────────────────┐
│  monitor.yourdomain.com (public or restricted)   │
└─────────────────────────────────────────────────┘
```

### Components

| File | Purpose |
|------|---------|
| `server.mjs` | HTTP server — serves dashboard + `/api/metrics` endpoint |
| `dashboard.html` | Single-page dashboard UI (vanilla HTML/CSS/JS) |
| `parse-activity.mjs` | Parses OpenClaw session `.jsonl` files for agent activity |
| `parse-costs.mjs` | Estimates LLM costs from session logs using token counts |
| `node-metrics.mjs` | Lightweight metrics agent for remote VPS nodes (optional) |
| `metrics.json` | Static fallback / cache file |

---

## Prerequisites

- **Node.js** v18+ (v22 recommended)
- **OpenClaw** installed and running (the dashboard reads its session logs)
- **SSH access** to any remote Mac nodes (for metrics collection via SSH)
- **Cloudflare Tunnel** (optional — for exposing the dashboard publicly)

---

## Setup

### 1. Create the monitoring directory

```bash
mkdir -p ~/.openclaw/workspace/monitoring
cd ~/.openclaw/workspace/monitoring
```

### 2. Configure server.mjs

Copy `server.mjs` to your monitoring directory. Edit the `NODES` object to match your infrastructure:

```javascript
const NODES = {
  // Local hub machine (no SSH needed)
  myhub: {
    host: null, port: null, type: 'hub',
    specs: 'M1 Max · 64GB · 2TB', cost: 0,
    dns: null, ip: 'local',
    agents: ['AgentA', 'AgentB']
  },

  // Remote Mac node (metrics collected via SSH)
  mymacmini: {
    host: 'macmini.local', port: null, type: 'node',
    specs: 'M4 · 16GB · 256GB', cost: 0,
    dns: 'macmini.local', ip: 'local',
    agents: ['AgentC']
  },

  // Remote VPS (runs node-metrics.mjs on port 9100)
  myvps: {
    host: '1.2.3.4', port: 9100, type: 'vps',
    specs: '4 vCPU · 8GB · 160GB', cost: 48,
    dns: 'vps.example.com', ip: '1.2.3.4',
    agents: ['AgentD']
  },
};
```

**Node types:**
- `hub` — the local machine running OpenClaw (metrics collected locally via `top`/`vm_stat`/`df`)
- `node` — a paired Mac node (metrics collected via SSH, must have passwordless SSH configured)
- `vps` — a remote Linux VPS (runs `node-metrics.mjs` as a metrics agent on port 9100)

### 3. Configure subscriptions (optional)

Edit the `SUBSCRIPTIONS` object in `server.mjs` to reflect your actual costs:

```javascript
const SUBSCRIPTIONS = {
  'Claude Max':          { cost: 200, category: 'ai', provider: 'anthropic' },
  'OpenAI Plus':         { cost: 20,  category: 'ai', provider: 'openai' },
  'DigitalOcean VPS':    { cost: 24,  category: 'hosting', detail: '2 vCPU / 4GB' },
  'Home Electricity':    { cost: 3,   category: 'infra', detail: 'estimated for always-on Mac' },
};
```

### 4. Configure cost parsing

`parse-costs.mjs` reads OpenClaw session `.jsonl` files to estimate LLM spend. It uses standard token pricing:

```javascript
// Default pricing (edit to match your plan)
const PRICING = {
  'claude-opus-4': { input: 15, output: 75, cacheRead: 1.5, cacheWrite: 3.75 },
  'claude-sonnet-4': { input: 3, output: 15, cacheRead: 0.3, cacheWrite: 0.75 },
  // ... per million tokens
};
```

The parser scans:
```
~/.openclaw/agents/*/sessions/*.jsonl
```

### 5. Set up the activity parser

`parse-activity.mjs` reads session logs to track:
- Last active timestamp per agent
- Message counts
- Tool call counts
- Active session count

It scans the same session directory structure as the cost parser.

### 6. Deploy node-metrics.mjs to VPS nodes (optional)

If you have remote Linux/VPS nodes, deploy the lightweight metrics agent:

```bash
# On the VPS
scp node-metrics.mjs user@vps:/opt/openclaw/
ssh user@vps "node /opt/openclaw/node-metrics.mjs &"

# Or as a systemd service:
cat > /etc/systemd/system/openclaw-metrics.service << EOF
[Unit]
Description=OpenClaw Node Metrics
After=network.target

[Service]
Type=simple
ExecStart=/usr/bin/node /opt/openclaw/node-metrics.mjs
Restart=always

[Install]
WantedBy=multi-user.target
EOF

systemctl enable --now openclaw-metrics
```

This serves system metrics on port 9100 (`GET /metrics`) and accepts call tracking (`POST /track`).

---

## Running the Dashboard

### Quick start (foreground)

```bash
node ~/.openclaw/workspace/monitoring/server.mjs
# Dashboard on http://127.0.0.1:8877
```

### As a macOS launchd service (recommended)

Create the plist file:

```bash
cat > ~/Library/LaunchAgents/ai.openclaw.monitor.plist << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key>
  <string>ai.openclaw.monitor</string>
  <key>ProgramArguments</key>
  <array>
    <string>/path/to/node</string>
    <string>/path/to/.openclaw/workspace/monitoring/server.mjs</string>
  </array>
  <key>RunAtLoad</key>
  <true/>
  <key>KeepAlive</key>
  <true/>
  <key>StandardOutPath</key>
  <string>/tmp/openclaw-monitor.log</string>
  <key>StandardErrorPath</key>
  <string>/tmp/openclaw-monitor.log</string>
</dict>
</plist>
EOF
```

Load it:

```bash
launchctl load ~/Library/LaunchAgents/ai.openclaw.monitor.plist
```

Replace `/path/to/node` with your actual Node.js binary path (e.g., `which node`).

### As a Linux systemd service

```bash
cat > /etc/systemd/system/openclaw-dashboard.service << 'EOF'
[Unit]
Description=OpenClaw Dashboard
After=network.target

[Service]
Type=simple
User=youruser
ExecStart=/usr/bin/node /home/youruser/.openclaw/workspace/monitoring/server.mjs
Restart=always
Environment=HOME=/home/youruser

[Install]
WantedBy=multi-user.target
EOF

systemctl enable --now openclaw-dashboard
```

---

## Exposing Publicly (Cloudflare Tunnel)

The dashboard binds to `127.0.0.1:8877` by default. To expose it via a domain:

### 1. Install cloudflared

```bash
# macOS
brew install cloudflared

# Linux
curl -L https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64 -o /usr/local/bin/cloudflared
chmod +x /usr/local/bin/cloudflared
```

### 2. Create a tunnel

```bash
cloudflared tunnel login
cloudflared tunnel create openclaw-tunnel
```

### 3. Configure the tunnel

Create `~/.cloudflared/config.yml`:

```yaml
tunnel: <your-tunnel-id>
credentials-file: ~/.cloudflared/<your-tunnel-id>.json

ingress:
  - hostname: monitor.yourdomain.com
    service: http://127.0.0.1:8877
  - service: http_status:404
```

### 4. Set up DNS

```bash
cloudflared tunnel route dns openclaw-tunnel monitor.yourdomain.com
```

### 5. Run the tunnel as a service

**macOS (launchd):**

```bash
cat > ~/Library/LaunchAgents/ai.openclaw.cloudflared.plist << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key>
  <string>ai.openclaw.cloudflared</string>
  <key>ProgramArguments</key>
  <array>
    <string>/opt/homebrew/bin/cloudflared</string>
    <string>--config</string>
    <string>/path/to/.cloudflared/config.yml</string>
    <string>tunnel</string>
    <string>run</string>
    <string>openclaw-tunnel</string>
  </array>
  <key>RunAtLoad</key>
  <true/>
  <key>KeepAlive</key>
  <true/>
  <key>StandardOutPath</key>
  <string>~/Library/Logs/cloudflared.out.log</string>
  <key>StandardErrorPath</key>
  <string>~/Library/Logs/cloudflared.err.log</string>
</dict>
</plist>
EOF

launchctl load ~/Library/LaunchAgents/ai.openclaw.cloudflared.plist
```

**Linux (systemd):**

```bash
cloudflared service install
systemctl enable --now cloudflared
```

---

## API

### `GET /api/metrics`

Returns a JSON payload with all dashboard data:

```json
{
  "timestamp": "2026-03-05T12:00:00.000Z",
  "nodes": [
    {
      "name": "macmax",
      "type": "hub",
      "specs": "M1 Max · 64GB · 2TB",
      "cpu_pct": 12.5,
      "mem_total_mb": 65536,
      "mem_used_mb": 32000,
      "disk_total_gb": 1862,
      "disk_used_gb": 950,
      "uptime": "14 days",
      "agents": ["Patoo", "Calypso"],
      "connected": true
    }
  ],
  "agents": {
    "patoo": {
      "lastActive": "2026-03-05T11:59:00.000Z",
      "messages": 42,
      "toolCalls": 156,
      "sessions": 3
    }
  },
  "costs": {
    "today": 4.23,
    "week": 28.50,
    "month": 112.00,
    "byAgent": { "patoo": 45.00, "vandana": 34.79 }
  },
  "subscriptions": {
    "Claude Max": { "cost": 200, "category": "ai" }
  },
  "total_monthly_cost": 318.32
}
```

---

## Customization

### Adding new nodes

Add entries to the `NODES` object in `server.mjs`. Supported types:
- **hub** — local machine, metrics via `top`/`vm_stat`/`df`
- **node** — remote Mac, metrics via SSH (requires passwordless SSH keys)
- **vps** — remote Linux, metrics via HTTP from `node-metrics.mjs` on port 9100

### Modifying the dashboard UI

Edit `dashboard.html` directly — it's a single self-contained HTML file with inline CSS and JS. No build step needed.

### Adding authentication

The dashboard has no built-in auth. Options:
- **Cloudflare Access** — add Zero Trust policies to your tunnel (recommended)
- **Basic auth** — add HTTP basic auth in `server.mjs`
- **VPN** — only expose on your local network

---

## Directory Structure

```
~/.openclaw/workspace/monitoring/
├── README.md              ← this file
├── server.mjs             ← main dashboard server (port 8877)
├── dashboard.html         ← single-page UI
├── parse-activity.mjs     ← agent activity parser
├── parse-costs.mjs        ← LLM cost estimator
├── node-metrics.mjs       ← remote VPS metrics agent
└── metrics.json           ← static metrics cache
```

---

## Troubleshooting

**Dashboard shows "offline" for a node:**
- For Mac nodes: check SSH connectivity (`ssh user@node.local uptime`)
- For VPS nodes: check `node-metrics.mjs` is running (`curl http://vps:9100/metrics`)
- Check OpenClaw node state: `cat ~/.openclaw/state/nodes.json`

**Cost data not showing:**
- Ensure session `.jsonl` files exist in `~/.openclaw/agents/*/sessions/`
- Check `parse-costs.mjs` runs without errors: `node parse-costs.mjs`

**Activity data stale:**
- Activity is cached for 60s, costs for 5 min
- Check `parse-activity.mjs` runs: `node parse-activity.mjs | head`

**Tunnel not connecting:**
- Check tunnel status: `cloudflared tunnel info openclaw-tunnel`
- Check logs: `tail -f ~/Library/Logs/cloudflared.err.log`

---

## Credits

Built by the [OpenClaw](https://github.com/openclaw/openclaw) agent team. Part of the OpenClaw multi-agent ecosystem.

- **Docs:** https://docs.openclaw.ai
- **Community:** https://discord.com/invite/clawd
- **Skills:** https://clawhub.com
