---
layout: post
title: "Cloudflare Blocking Headless Browser Logins from VPS IPs — and How I Got Past It"
date: 2026-07-13 14:00:00 +0200
categories: [troubleshooting, cloudflare, browser-automation]
tags: [cloudflare, headless-chrome, vps, playwright, browser-automation, bot-detection]
---

My AI agent needs to log into a few web services every week — checking dashboards, pulling reports, clicking through standard SaaS flows. Nothing shady. But the moment I pointed a headless Chromium at any site behind Cloudflare, I hit a wall. Not a soft wall — the full "Checking your browser before accessing" interstitial with the spinning Cloudflare logo, followed by a "Please enable JavaScript and cookies to continue" error. The agent couldn't click through it because the interstitial was served *before* the page even loaded. The browser never reached the login form.

The first time it happened I thought it was a timing bug. I added waits, retries, a full page reload. Same result every time. The second time I opened the page manually from the same VPS and got the same interstitial — that's when I realized this wasn't a code problem. It was a *reputation* problem.

## What Cloudflare Actually Detects

Cloudflare's bot detection (the "I'm Under Attack" / JS challenge mode) checks several things simultaneously:

1. **IP reputation** — your VPS IP is in a data-center range. Cloudflare knows Hetzner, DigitalOcean, Linode, OVH, and AWS blocks cold. They don't see residential traffic on those IPs, so they flag them by default.
2. **`navigator.webdriver`** — headless Chrome and headed Playwright chrome both set `navigator.webdriver = true` by default. Cloudflare checks this.
3. **User agent fingerprint** — headless Chrome has a distinct user-agent string that lacks the full set of Chrome version identifiers a normal install reports.
4. **Canvas and WebGL rendering** — headless mode uses a software renderer. The canvas fingerprint is subtly different — shorter draw buffers, no GPU vendor strings.
5. **Request patterns** — a headless browser loading a page with no prior cookies, no favicon request, no preconnect hints looks different from a real user.

Individually you can work around each. Together they add up to an instant block.

## What Did NOT Work For Me

Spending an afternoon trying things that failed:

```python
page.evaluate("Object.defineProperty(navigator, 'webdriver', {get: () => undefined})")
```

Doesn't work. Cloudflare evaluates `navigator.webdriver` before any page script runs. You can't patch it after the fact because the interstitial already triggered.

```bash
--user-agent="Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 ..."
```

Custom user agents don't help much either — Cloudflare does client-side behavior analysis, not just header matching. The canvas fingerprint and the WebGL differences give it away regardless of what user-agent string you send.

Using a proxy (residential or mobile IP) through a service like BrightData is the "just pay for it" solution, but it adds latency, a monthly bill, and a dependency — plus many residential proxies recycle IPs quickly enough that Cloudflare still challenges you after a few requests.

## What Actually Works

### 1. Run headed, not headless — with a virtual display

The single biggest change: **don't pass `--headless`**. Instead, run Chrome with `Xvfb` as a virtual frame buffer. The browser thinks it's displaying on a real screen. Cloudflare sees a real rendering pipeline, real GPU calls (through the software renderer that Xvfb provides), and real compositing.

```bash
# Install Xvfb
apt install xvfb

# Start a virtual display
Xvfb :99 -screen 0 1920x1080x24 &

# Launch Chrome without --headless, pointing at the virtual display
DISPLAY=:99 google-chrome --no-sandbox --disable-dev-shm-usage
```

Chrome running under Xvfb produces different canvas fingerprints than `--headless` mode — enough to get past the first Cloudflare layer. It's not perfect (you'll still get challenged occasionally), but it gets through the JS challenge successfully 90%+ of the time, where headless mode fails 100%.

I wrapped this into a launch script:

```bash
#!/bin/bash
# launch-chrome-for-agent.sh — run Chrome headed into a virtual display

XVFB_DISPLAY=${XVFB_DISPLAY:-:99}
CHROME_USER_DIR=${CHROME_USER_DIR:-$HOME/.config/chrome-agent}

# Start Xvfb if not running
pgrep -x Xvfb > /dev/null || Xvfb $XVFB_DISPLAY -screen 0 1920x1080x24 &

# Wait for Xvfb to be ready
sleep 1

export DISPLAY=$XVFB_DISPLAY

exec /usr/bin/google-chrome \
  --remote-debugging-port=9222 \
  --user-data-dir="$CHROME_USER_DIR" \
  --no-sandbox \
  --disable-dev-shm-usage \
  --disable-gpu \
  --disable-software-rasterizer \
  --disable-features=ChromeWhatsNewUI \
  "$@"
```

### 2. Patch `navigator.webdriver` at launch, not runtime

The only reliable way to suppress the webdriver flag is through Chrome's command-line args *before* the process starts:

```bash
--disable-blink-features=AutomationControlled
```

This removes the `navigator.webdriver` property entirely instead of trying to hide it. You need to set it at launch — late-bound page.evaluate patches are pointless.

### 3. Accept the JS challenge

Cloudflare's JS challenge is a tiny JavaScript puzzle that the browser has to solve before it can load the real page. Headless Chrome in `--headless` mode often fails to run this puzzle correctly (it times out or the WebGL calls return null). Headed Chrome under Xvfb runs the puzzle fine. But your agent still needs to **wait for it to complete** before trying to interact with the page.

The pattern I use in Playwright-based agents:

```python
# Navigate
page.goto("https://target-site.com/login", wait_until="domcontentloaded")

# Wait for Cloudflare challenge to pass (up to 15s)
try:
    page.wait_for_selector("#main-content", timeout=15000)
except:
    page.wait_for_url("**/login**", timeout=10000)

# Now the real page is loaded — proceed
page.fill("#username", "myuser")
```

The key insight: don't wait for `networkidle` on the initial navigation — the interstitial might keep the network busy. Wait for `domcontentloaded` and then wait for a specific element that only exists on the real page.

### 4. Warm up the IP

A cold VPS IP (never visited the site) gets the hardest challenge. After the first successful visit, Cloudflare sets a `cf_clearance` cookie that lasts a few hours. I have the agent visit each target site once during startup (before the real task) to generate this cookie, so the actual login flow doesn't hit the interstitial.

## The Truth: You Can't Fully Hide a VPS IP

All of the above gets you past the JS challenge into the "medium risk" bucket. Cloudflare will still show a captcha or block if you look too automated — too many requests, too fast, too many sites from the same IP. I run this agent on a $6/month VPS and I've accepted that I'll get blocked from a new site about 1 in 10 times on the first try. A retry 15 minutes later usually works (the IP cools down).

If you need 100% reliable unauthenticated scraping behind Cloudflare, you need a residential proxy service and it costs real money. If you need to log into a handful of sites you already have credentials for, the Xvfb + `--disable-blink-features=AutomationControlled` approach above works well enough for daily use. I've been running it for two months and the only blocks have been on sites I'd never visited before — everything with a warm `cf_clearance` cookie goes through.

---

*Getting past Cloudflare's detection on a VPS is the kind of edge case that takes hours of trial and error to solve. If you want a ready-to-go agent stack with these configs baked in, I maintain the setup I use daily at <https://brainscratch.gumroad.com/l/snjbhd>.*