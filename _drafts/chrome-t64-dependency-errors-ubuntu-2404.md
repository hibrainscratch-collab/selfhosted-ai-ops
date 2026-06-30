---
layout: post
title: "Installing Chrome on Ubuntu 24.04: The t64 Dependency Trap"
date: 2026-06-30 18:19:00 +0200
categories: [troubleshooting, chrome, ubuntu]
tags: [chrome, ubuntu-2404, dpkg, apt, dependency-resolution]
---

The first time I downloaded the `google-chrome-stable_current_amd64.deb` on a fresh Ubuntu 24.04 server and ran `dpkg -i`, I got back a wall of "dependency problems prevent configuration" errors. Twelve missing packages, all at once. I did what any reasonable person does: tried `apt install` with the explicit names, hit a "virtual package" ambiguity, tried `--fix-broken`, and eventually watched 142 packages get pulled in. It's annoying enough that I'm writing it down so future-me doesn't have to rediscover it.

## The Error

After downloading and running dpkg:

```bash
wget https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb
dpkg -i google-chrome-stable_current_amd64.deb
```

dpkg reports success but then apt refuses to configure Chrome:

```
dpkg: dependency problems prevent configuration of google-chrome-stable:
 google-chrome-stable depends on fonts-liberation; but:
  Package fonts-liberation is not installed.
 google-chrome-stable depends on libasound2; but:
  Package libasound2 is not installed.
 google-chrome-stable depends on libatk-bridge2.0-0; but:
  Package libatk-bridge2.0-0 is not installed.
 google-chrome-stable depends on libatk1.0-0; but:
  Package libatk1.0-0 is not installed.
 google-chrome-stable depends on libatspi2.0-0; but:
  Package libatspi2.0-0 is not installed.
 google-chrome-stable depends on libcups2; but:
  Package libcups2 is not installed.
 google-chrome-stable depends on libgbm1; but:
  Package libgbm1 is not installed.
 google-chrome-stable depends on libgtk-3-0; but:
  Package libgtk-3-0 is not installed.
 google-chrome-stable depends on libpango-1.0-0; but:
  Package libpango-1.0-0 is not installed.
 google-chrome-stable depends on libcairo2; but:
  Package libcairo2 is not installed.
 google-chrome-stable depends on libvulkan1; but:
  Package libvulkan1 is not installed.
 google-chrome-stable depends on xdg-utils; but:
  Package xdg-utils is not installed.
```

Twelve dependencies, all missing. Standard enough for a bare-bones cloud image — but the fix isn't as straightforward as it should be.

## What's Actually Happening

Ubuntu 24.04 ("Noble") shipped with a mass rename of shared libraries to include a `t64` suffix — this was the big 64-bit `time_t` migration. Packages that were `libasound2` are now `libasound2t64`. The old package names still exist as transitional dummy packages that depend on the new `t64` names, but the transition isn't seamless.

When you run `dpkg -i` on Chrome's `.deb`, it records its dependency list using the **pre-Noble** package names (`libasound2`, not `libasound2t64`). Then when `apt-get install -f` tries to satisfy those deps, it can find the transitional packages in most cases — but `libasound2` is a **virtual package**, which creates ambiguity.

## The Fix (real steps that worked)

### Step 1: Try explicit install — hits the virtual package wall

```bash
apt-get install -y fonts-liberation libasound2 libatk-bridge2.0-0 libatk1.0-0 \
  libatspi2.0-0 libcups2 libgbm1 libgtk-3-0 libpango-1.0-0 libcairo2 \
  libvulkan1 xdg-utils
```

This fails with:

```
Package libasound2 is a virtual package provided by:
  libasound2t64 1.2.11-1build2
You should explicitly select one to install.
```

`libasound2` is a virtual package on Noble — it's provided by `libasound2t64`, but apt refuses to resolve it automatically when given as an explicit target. The same ambiguity doesn't apply to the other packages (they're real transitional packages, not virtual), but one blocker is enough to kill the whole command.

### Step 2: Fix the asound ambiguity, then let apt do the rest

Replace `libasound2` with `libasound2t64` in the install list, then let `--fix-broken` pull in the rest:

```bash
apt install libasound2t64 -y
apt --fix-broken install -y
```

That second command pulls in **142 packages** — including all the other listed deps and their transitive dependencies, many of which are the `t64`-renamed variants (`libcups2t64`, `libatk1.0-0t64`, etc.). After this, `dpkg --configure -a` completes and Chrome is installed and working.

### Clean one-liner for a fresh server

```bash
wget -q https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb && \
  dpkg -i google-chrome-stable_current_amd64.deb 2>/dev/null; \
  apt-get install -y libasound2t64 && \
  apt --fix-broken install -y
```

Ignore the dpkg errors on the first line. The `libasound2t64` + `--fix-broken` combo handles everything.

## Why This Matters for Self-Hosted AI Ops

If you're running headless browser automation on a VPS — whether for Playwright, Puppeteer, browser-based agents, or plain old web scraping — Chrome on a Ubuntu 24.04 server is table stakes. And VPS providers are shipping Noble by default now. You *will* hit this dependency wall.

The good news: once the deps are sorted, Chrome runs fine. There's no deeper compatibility issue. It's just a packaging name mismatch that costs you 15 minutes the first time.

---

*If you're fighting these same battles setting up automated browser agents on a VPS, I've been collecting these fixes and templates into a practical ops pack. [Check out the Self-Hosted AI Assistant Ops Kit on Gumroad](https://brainscratch.gumroad.com/l/snjbhd).*
