---
layout: post
title: "Fighting Chrome on Ubuntu 24.04: The t64 Dependency Trap"
date: 2026-06-30 18:19:00 +0200
categories: [troubleshooting, chrome, ubuntu]
tags: [chrome, ubuntu-2404, playwright, browser-automation, headless]
---

The first time I ran `playwright install chromium` on a fresh Ubuntu 24.04 server and watched it fail with a wall of red text about "dependencies cannot be satisfied," I did what any reasonable person does: tried `--force`, tried `apt --fix-broken install`, tried restarting, tried Googling the exact error message, and then sat there wondering if I should just switch to Firefox.

Spoiler: I didn't switch. But the fix is annoying enough that I'm writing it down so future-me doesn't have to rediscover it.

## The Error

Here's the actual output that greeted me:

```
Error: Failed to install chromium
Ubuntu 24.04 LTS (Noble Numbat) is not officially supported by Playwright.
Please use Ubuntu 22.04 or Debian 11/12 instead.
```

Playwright's system-dependency installer (`playwright install-deps chromium`) then proceeds to try anyway. It runs `apt-get install` with a long list of packages like `libnss3`, `libnspr4`, `libatk1.0-0t64`, `libcups2t64`, and so on. And *that's* where things fall apart, because on 24.04 several of those packages have been renamed with a `t64` suffix as part of Ubuntu's 64-bit time_t transition.

The key error from apt reads something like this:

```
The following packages have unmet dependencies:
 libnss3 : Depends: libnspr4 (>= 2:4.12) but it is not installable
E: Unable to correct problems, you have held broken packages.
```

## What's Actually Happening

Ubuntu 24.04 ("Noble") shipped with a mass rename of shared libraries to include a `t64` suffix — this was the big 64-bit `time_t` migration. Packages that were `libfoo.so.0` became `libfoo.so.0t64`. The old package names still exist as transitional dummy packages that depend on the new `t64` names, but Playwright's internal dependency resolver doesn't know about this yet. It generates an apt command referencing the old non-t64 names, and since the transitional packages aren't always direct replacements, apt bails.

The real issue is that Playwright's `install-deps` subcommand (as of mid-2025/early-2026) hardcodes the pre-Noble package names. It hasn't been updated to use the `t64` variants Ubuntu now ships.

## The Fix

I've landed on two approaches, both of which work.

### Approach 1: Install the system deps manually

Let Playwright install the browser binary but handle system dependencies yourself:

```bash
sudo apt-get update
sudo apt-get install -y libnss3 libnspr4 libatk1.0-0t64 libcups2t64 \
  libdrm2 libdbus-1-3 libxkbcommon0 libxcomposite1 libxdamage1 \
  libxrandr2 libgbm1 libpango-1.0-0 libcairo2 libasound2t64 \
  libatspi2.0-0t64 libwayland-client0
```

Notice I mixed `t64` and non-t64 names — some packages got the rename, some didn't. The `t64` ones are specifically those that declare a `t64` ABI variant. Package names ending in `t64` are the correct ones for Noble.

If you get a "package not found" for a t64 name, the non-t64 transitional package still works on Noble and pulls in the t64 variant through dependencies. The trick is knowing which is which, which brings me to approach 2.

### Approach 2: Pin to Jammy Debs (Hacky, but works)

If you don't care about elegance and just want Chrome to run:

```bash
# Add focal/jammy repos for the specific chrome deps
# Then apt install from those repos
```

I don't actually recommend this — it creates a FrankenDebian situation. Stick with approach 1.

### Approach 3: Use the `--force` flag with a dep whitelist

Playwright 1.50+ added a `--skip-browser-download` flag. Combine with manual dependency installation:

```bash
npx playwright install --with-deps --force chromium
```

This ignores Playwright's distro check and tries to install deps anyway, but on 24.04 it still fails because of the t64 mismatch. So this doesn't save you.

## The Cleanest Solution

What I actually do now on every 24.04 server:

```bash
# Let Playwright download the browser binary
npx playwright install chromium

# If the system dep install stage fails, do it by hand
sudo apt-get install -y $(dpkg -l | grep -oP 'lib\S+t64' | sort -u) \
  libnss3 libnspr4 libatk-bridge2.0-0 libcups2 \
  libxshmfence1 libglib2.0-0 libgtk-3-0

# Verify it works
npx playwright open chromium
```

If you get `error while loading shared libraries: libnss3.so: cannot open shared object file`, you're still missing a dep. Run `ldd /path/to/chrome | grep "not found"` to see exactly which ones.

## Why This Matters for Self-Hosted AI Ops

If you're running headless browser automation for AI operations — scraping LLM leaderboards, automating login flows for API dashboards, or running browser-based agents — Chrome on a VPS is table stakes. And VPS providers are shipping Ubuntu 24.04 by default now. You *will* hit this.

The good news: once the deps are sorted, Chrome runs fine. Performance is stable. There's no deeper compatibility issue. It's just a packaging name mismatch that costs you 20 minutes of hair-pulling the first time.

## The One-Liner I Wish I'd Had

For a fresh 24.04 install where you want Playwright + Chromium working in under a minute:

```bash
apt-get update && apt-get install -y ca-certificates curl gnupg && \
npx playwright install chromium 2>/dev/null || true && \
apt-get install -y libnss3 libnspr4 libatk1.0-0t64 libcups2t64 \
  libdrm2 libxkbcommon0 libxcomposite1 libxdamage1 libgbm1 \
  libasound2t64 libatspi2.0-0t64 2>/dev/null && \
npx playwright install chromium
```

Ignore the errors on the first attempt. The second one will work.

---

*If you're fighting these same battles setting up automated browser agents on a VPS, I've been collecting these fixes into a single Ansible-style playbook. [Check out the self-hosted AI agent starter kit on Gumroad](https://brainscratch.gumroad.com/l/snjbhd).*