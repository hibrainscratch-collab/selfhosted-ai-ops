---
layout: post
title: "Chrome/t64 Dependency Errors on Ubuntu 24.04 — What I've Learned Since June"
date: 2026-08-03 09:00:00 -0500
categories: [openclaw, ubuntu, selfhosted, troubleshooting]
tags: [ubuntu, chrome, dependencies, t64, headless-browser, openclaw]
---

Back in June I wrote up how I got Chrome running on Ubuntu 24.04 for OpenClaw's browser tooling despite the `t64` transition breaking half the package tree. I thought I'd solved it. And I had — for about three weeks. Then the August `apt` updates hit, and I learned how incomplete my fix really was.

Here's the updated story, including the error messages that sent me back to the drawing board, and the more robust setup that's been stable since.

## The Part I Got Right

The core problem hasn't changed: Ubuntu 24.04 migrated several core libraries to the `t64` (64-bit time_t) ABI. Packages like `libssl3`, `libffi8`, and `libc6-dev` got renamed or split. Chrome's `.deb` still depends on the old names. The June fix was brute-force — grab the old-ABI compatibility packages and pin them:

```bash
sudo apt install libssl3t64 libffi8 libnss3
```

That got Chrome installed and running. But I only tested with `google-chrome-stable --version` and one simple page load. I didn't do extended sessions.

## The August Breakage

Three weeks of quiet operation, then I started seeing this in OpenClaw's browser logs:

```
Error: Failed to launch the browser process!
/opt/google/chrome/chrome: error while loading shared libraries:
libnss3.so: cannot open shared object file: No such file or directory
```

Running `ldd /opt/google/chrome/chrome | grep "not found"` showed four missing libraries. The June update had bumped `libnss3` and friends to newer builds that weren't backward-compatible with Chrome's pinned version. Running `apt upgrade` had silently replaced my compat packages.

The real fix wasn't pinning versions. It was isolating Chrome from the system package manager entirely.

## The Docker-CUPS Approach

I tried the clean solution first: install Chrome inside a minimal Docker container and proxy the browser through that. The browser tooling in OpenClaw supports a remote debugging port, so this should be straightforward:

```bash
docker run -d --name chrome-headless \
  -p 9222:9222 \
  ghcr.io/chromium/chromium:latest \
  --headless --remote-debugging-port=9222 --no-sandbox
```

And it *did* work — for about a day. The container kept OOM-killing itself on complex pages (Grafana dashboards, heavy JS SPAs). The default `--memory=512m` was nowhere near enough, and even bumping it to 2GB felt wasteful for what's supposed to be a lightweight automation tool running on a $10 VPS.

## The Flatpak Alternative

Next I tried the Flatpak Chrome install. Flatpak bundles its own dependency tree, sidestepping the t64 issue entirely:

```bash
flatpak install flathub com.google.Chrome
flatpak run com.google.Chrome --headless --remote-debugging-port=9222
```

This worked. The Flatpak runtime provides whatever old libraries Chrome needs. But there's a catch: Flatpak Chrome with `--headless` doesn't render WebGL or canvas-heavy content the same way as the system Chrome. Some OpenClaw page snapshots came back blank or with "Aw, Snap!" errors. Not acceptable for reliable automation.

## What Actually Fixed It

I ended up back with a system install, but with a much more careful approach:

1. **Installed from the Google repository, not a downloaded `.deb`.** The repo version tracks Chrome's own dependencies more precisely.

```bash
curl -fsSL https://dl-ssl.google.com/linux/linux_signing_key.pub | sudo gpg --dearmor -o /usr/share/keyrings/google-chrome.gpg
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/google-chrome.gpg] http://dl.google.com/linux/chrome/deb/ stable main" | sudo tee /etc/apt/sources.list.d/google-chrome.list
```

2. **Installed the `t64` compat layer properly**, not just the three libraries I grabbed in June:

```bash
sudo apt install libssl3t64 libssl-dev libnss3 libnss3-tools libnspr4 libavcodec-extra libavformat-extra libavutil-extra
```

3. **Created an apt preferences file to hold Chrome's critical dependencies**:

```bash
cat << 'EOF' | sudo tee /etc/apt/preferences.d/pin-chrome-deps
Package: libnss3 libnspr4 libssl3t64 libavcodec-extra libavformat-extra
Pin: version *
Pin-Priority: 1001
EOF
```

4. **Set up a post-update check script** that runs after every `apt upgrade`:

```bash
#!/bin/bash
# /usr/local/bin/check-chrome-deps
MISSING=$(ldd /opt/google/chrome/chrome 2>/dev/null | grep "not found" | wc -l)
if [ "$MISSING" -gt 0 ]; then
  echo "WARNING: Chrome has $MISSING unresolved library dependencies"
  notify-send -u critical "Chrome deps broken after update"
fi
```

Linked into `/etc/apt/apt.conf.d/99check-chrome` so it runs automatically.

## The Actual Test That Gives Me Confidence

The real verification isn't `chrome --version`. It's running OpenClaw's full browser flow against a real target. I wrote a tiny smoke test:

```bash
google-chrome-stable --headless --no-sandbox --disable-gpu \
  --dump-dom https://example.com 2>/dev/null | grep -q "Example Domain" && \
  echo "CHROME_OK" || echo "CHROME_FAIL"
```

And then a heavier test that loads a page with JavaScript rendering:

```bash
google-chrome-stable --headless --no-sandbox --disable-gpu \
  --screenshot=/tmp/test.png --window-size=1920,1080 \
  https://brainscratch.gumroad.com/l/snjbhd 2>/dev/null && \
  identify /tmp/test.png | grep -q "1920x1080" && echo "RENDER_OK"
```

If both pass, the browser setup is solid. If the screenshot test fails, something's wrong with the rendering pipeline — usually a missing libavcodec dependency.

## What I'd Do Differently

If I were setting this up fresh today, I'd skip the direct `.deb` download entirely and go straight to the Google repo + pinned dependencies. The Flatpak route is worth trying if you're on a desktop where rendering fidelity doesn't matter, but for headless VPS automation, the native install with proper pinning is still the most reliable path — you just have to do the post-update check to catch regressions early.

The root lesson isn't about Chrome or t64 specifically. It's about treating OS package upgrades as a risk vector for your agent tooling, not an afterthought. Your browser is the most complex dependency in your automation stack. Treat it like one.

---

*Running your own self-hosted ops agent? I've put together a complete OpenClaw setup with pre-configured tools, guardrails, and browser automation that handles all of this. [Check it out here](https://brainscratch.gumroad.com/l/snjbhd).*