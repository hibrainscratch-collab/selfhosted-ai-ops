---
layout: post
title: "Self-Hosting OpenClaw: systemd, Caddy, and Keeping Your Agent Alive"
date: 2026-08-17 09:00:00 -0500
categories: [infrastructure, openclaw, devops]
tags: [openclaw, systemd, caddy, reverse-proxy, self-hosted, linux-administration]
---

I've been running OpenClaw as my primary AI agent interface for a few months now, and the single biggest reliability win was getting it set up properly behind systemd with a Caddy reverse proxy. Out of the box, OpenClaw's gateway runs as a foreground process on a local port. That works fine for tinkering, but if you want your agent available when you need it — without logging into a tmux session every time your VPS reboots — you need infrastructure. Here's exactly what I did.

## The Problem

After the first couple of server reboots, I realized running the gateway by hand meant I'd SSH in, find the process was gone, start it up again, and feel like I was house-sitting a server instead of using one. The gateway exposes a WebSocket-based API that the OpenClaw client connects to, and on a VPS you also need TLS termination — browsers won't connect to a bare WebSocket over HTTP, and sending your API keys over plaintext is a non-starter.

## Step 1: systemd Service Unit

I created `/etc/systemd/system/openclaw-gateway.service`:

```ini
[Unit]
Description=OpenClaw Gateway Service
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=openclaw
Group=openclaw
WorkingDirectory=/opt/openclaw
ExecStart=/usr/bin/openclaw gateway start
Restart=always
RestartSec=5
StandardOutput=append:/var/log/openclaw/gateway.log
StandardError=append:/var/log/openclaw/gateway.log
Environment="NODE_ENV=production"
Environment="OPENCLAW_HOME=/opt/openclaw-data"

[Install]
WantedBy=multi-user.target
```

Key choices:

- **`Restart=always` with `RestartSec=5`** — if the gateway crashes or gets OOM-killed, it comes back. I've seen this happen once when the VPS was under memory pressure from a rogue Chromium instance (lesson: set Chrome's `--max_old_space_size`).
- **`After=network-online.target`** — ensures the network stack is fully ready before the gateway tries to bind. Without this, I occasionally saw bind errors on boot.
- **Dedicated `openclaw` user** — never run agent infrastructure as root. The `openclaw` user owns only what it needs.

I also set up log rotation so the journal doesn't eat all the disk:

```bash
# /etc/logrotate.d/openclaw
/var/log/openclaw/gateway.log {
    daily
    rotate 14
    compress
    delaycompress
    missingok
    notifempty
    create 0640 openclaw openclaw
}
```

## Step 2: Caddy for TLS Termination

OpenClaw's gateway binds locally (say, `127.0.0.1:44321`). Caddy handles the public-facing HTTPS and reverse-proxies to it. I'm using Caddy over nginx because automatic Let's Encrypt certificates are built in — no certbot, no cron for renewal, it just works.

`/etc/caddy/Caddyfile`:

```
agent.example.com {
    reverse_proxy 127.0.0.1:44321 {
        header_up X-Forwarded-For {remote_host}
        header_up X-Forwarded-Proto {scheme}
    }
}
```

That's it. Three lines. Caddy detects the WebSocket upgrade headers automatically and proxies them correctly — the OpenClaw client negotiates over HTTPS and then upgrades to a persistent WebSocket connection.

A few gotchas I hit:

1. **Caddy must serve on port 443.** If you're behind NAT or a firewall, make sure 443 is open. Caddy's HTTP challenge won't work otherwise.
2. **The OpenClaw client connects to `wss://agent.example.com`, not `https://`.** It's a WebSocket URL. If you test with `curl` it'll work for the initial handshake but the agent uses a persistent WS connection after that.
3. **Firewall rules.** I blocked everything except 443 (Caddy) and 22 (SSH). The gateway itself listens only on localhost, so it's never directly exposed.

## Step 3: Health Checking

The gateway has a built-in `/health` endpoint. I threw together a simple cron-based check that emails me if it goes missing:

```bash
#!/bin/bash
# /usr/local/bin/check-openclaw
curl -sf -o /dev/null http://127.0.0.1:44321/health || systemctl restart openclaw-gateway
```

Cron'd every 5 minutes. In practice, it's never fired — the gateway has been rock-solid since the systemd setup.

## The Actual Impact

Since I set this up:

- **Zero manual restarts.** The gateway has survived kernel updates, OOM events, and accidental reboots from failing to read the `-y` flag in apt.
- **The agent is always available.** I can send messages from my phone, walk away, come back hours later — it's still there.
- **TLS is handled for free.** Caddy rotates certs automatically, and I haven't touched a certificate file since day one.

## What I'd Do Differently

If I were setting this up fresh today, I'd put Caddy's config in version control (it's just a text file) and use a `docker-compose.yml` that bundles the gateway and Caddy together. The Docker approach eliminates most of the filesystem-level concerns — logs, permissions, user creation — at the cost of a slightly steeper initial setup. For a single-VPS deployment, systemd + bare Caddy is simpler and has fewer moving parts.

---

*If you're running your own AI agent infrastructure and want a cleaner setup than duct-taping together tmux sessions and manual SSH commands, I've been collecting these patterns into a practical ops pack. [Check out the Self-Hosted AI Assistant Ops Kit on Gumroad](https://brainscratch.gumroad.com/l/snjbhd).*