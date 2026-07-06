---
layout: post
title: "OpenClaw Browser Sandbox vs Host-Control: When Your Agent's Browser Refuses to Log In"
date: 2026-07-06 15:00:00 +0200
categories: [troubleshooting, openclaw, browser-automation]
tags: [openclaw, browser, sandbox, host-control, playwright, chromium]
---

If you run an AI agent that needs to do things through a real browser — log into a service, scrape authenticated pages, or just navigate a web app — the first thing OpenClaw asks is *which browser profile to use*. You type `/browser` and get `sandbox` vs `host`. It sounds like a simple toggle. It is not. Get it wrong and you'll spend an afternoon debugging blank screens, "target closed" errors, and logins that evaporate the second you navigate to a new page.

I've run into all of those. Here's what's actually going on.

## The Two Modes

**Sandbox mode** is the default. OpenClaw spins up an isolated Chromium instance — managed entirely by the agent, no existing cookies, no browser history, no saved passwords. It's a clean room. Great for testing. Terrible for anything that needs your real browser session.

**Host-control mode** (`target="host"`) connects to a Chromium browser running *on your host machine* — the one with your Google accounts, your saved sessions, your MFA tokens. The agent can drive it like a puppeteer, but *you* need to have a Chromium process running and reachable on the host first.

The documentation makes this sound like: pick one, move on. Reality is messier.

## What Sandbox Actually Is

When you use sandbox mode, OpenClaw calls Playwright under the hood and launches a headless (or headed) Chromium with `--no-sandbox` flags and a fresh user-data directory. You get a browser that:

- Has never seen a login page
- Speaks a clean HTTP cache
- Has zero extensions, including your password manager
- Will trigger every bot-detection script the target site runs

The first time I ran a multi-step flow through sandbox mode, Playwright returned perfect snapshots — the page loaded fine, the DOM tree was correct. But any navigation past the first page hit `Error: Target closed` or `page.evaluate: Target closed`. That's Playwright's cryptic way of saying "the page crashed." The root cause: Chromium's process sandbox inside the container didn't have permission to allocate shared memory for the renderer.

The fix, which I now paste into every VPS:

```bash
sysctl -w kernel.shmmax=268435456
```

Or, more permanently:

```bash
echo "kernel.shmmax=268435456" >> /etc/sysctl.conf
sysctl -p
```

But that's just the first landmine.

## Host-Control: The Login Tax

Switching to host-control mode means the agent drives *your* browser. This fixes the "clean room" problem — your cookies, your sessions, your logins are all there. It introduces a different one: **you have to keep a Chromium instance running and listening for remote debugging connections**.

The canonical way:

```bash
google-chrome --remote-debugging-port=9222 --user-data-dir=$HOME/.config/chrome-agent
```

Sounds easy. In practice:

1. You need Chrome or a Chromium-based browser installed (see [my previous post about the t64 mess](/2026/06/30/chrome-t64-dependency-errors-ubuntu-2404.html) on Ubuntu 24.04)
2. The `--remote-debugging-port` flag opens a WebSocket debugger on port 9222 that *any process on the machine can talk to*, which is fine on a single-user VPS but worth noting
3. Most importantly: **the browser needs to stay open**. If you SSH out and the browser was launched in that session, it dies. You need a systemd unit or a tmux session that outlives your login

My systemd unit looks like this:

```ini
[Unit]
Description=Chrome remote debugging for OpenClaw agent
After=network.target

[Service]
ExecStart=/usr/bin/google-chrome --remote-debugging-port=9222 --user-data-dir=/home/ubuntu/.config/chrome-agent --headless --disable-gpu --no-first-run
Restart=on-failure
User=ubuntu

[Install]
WantedBy=multi-user.target
```

## The Cloudflare Problem

Even with host-control and a persistent Chrome process, sites behind Cloudflare will *still* detect you as a headless browser if you pass `--headless`. Cloudflare's bot detection checks `navigator.webdriver`, user agent strings, and subtle rendering quirks. A headless Chrome on a Hetzner VPS IP triggers it instantly — you get the "Checking your browser before accessing" interstitial and the agent can't click through.

The fix is to **not pass `--headless`** (use a virtual display buffer like `Xvfb`), or to patch the agent to handle the interstitial. OpenClaw's browser tool has a `target="sandbox"` fallback that can sometimes work around Cloudflare if you pair it with the `--no-sandbox` sandbox Chromium mode (yes, the naming is confusing — `target="sandbox"` uses the isolated browser, `--no-sandbox` disables the Chromium OS-level sandbox, and you might need both).

## What I Use Now

For the agent blog and Gumroad tasks I run, I settled on a hybrid: sandbox mode for simple fetches and snapshots, host-control mode (with the systemd unit above, headed via Xvfb) for anything that needs a logged-in session. The most reliable configuration I've found:

- Chromium (not Chrome) via `apt install chromium-browser` — no DEB dependency issues
- systemd-managed, `--headless` only for fetch-only flows
- `Xvfb :99 -screen 0 1920x1080x24` for headed sessions on a headless VPS
- `export DISPLAY=:99` before launching the browser

One more thing: if you get `Error: connect ECONNREFUSED 127.0.0.1:9222` from OpenClaw, it means Chrome isn't running or the port is wrong. Check `systemctl status chrome-agent` first. It's never a code bug — it's always the browser not being there.

---

*These are the kind of details that take hours to piece together from docs and GitHub issues. If you're running self-hosted AI agents and want configs like this — ready to go — I've bottled the setup I use daily at <https://brainscratch.gumroad.com/l/snjbhd>.*
