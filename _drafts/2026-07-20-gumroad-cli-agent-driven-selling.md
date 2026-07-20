---
layout: post
title: "Setting Up the Gumroad CLI for Agent-Driven Selling"
date: 2026-07-20 18:00:00 +0200
categories: [automation, gumroad, selling]
tags: [gumroad, cli, agent, automation, self-hosted, digital-products]
---

I run a self-hosted AI agent that writes blog posts, drafts emails, and manages a small Gumroad shop. The shop part was always the odd one out — I could automate the writing and the publishing, but the product management side meant logging into a web UI, clicking through forms, and uploading files manually. If the whole point of having an agent is to not do things manually, that was a gap.

The Gumroad CLI changed that. It's a first-party terminal tool that covers everything from product creation to sales reports to refunds, and it's designed to work in scripts and automated pipelines. Getting it wired into an agent workflow took a few iterations. Here's what I learned.

## Installing the CLI

The Gumroad CLI is a Go binary distributed as a single executable. No npm, no pip, no Ruby gem — just download and run:

```bash
curl -fsSL https://gumroad.com/install-cli | bash
```

That drops a `gumroad` binary into `/usr/local/bin`. On a headless VPS, the install is instant — no dependencies, no build step, no conflicting package versions. First thing I checked:

```bash
gumroad version
```

And it worked. No errors, no missing libraries. That's rare enough in this space that I'll take the win.

## Authentication: The Device Flow

The CLI uses OAuth device authorization. The first time you run it, you need to authenticate with Gumroad:

```bash
gumroad auth login --no-input
```

This prints an approval URL. You open it in a browser, approve the request, and the CLI gets a token. On a desktop machine, `gumroad auth login --web` opens the browser automatically. On a headless VPS, you copy the URL, approve it from your laptop, and the CLI picks up the token.

The token gets stored in the system keyring or a local config file. After that, every command just works:

```bash
gumroad products list --json --no-input
```

Lists every product, with their IDs, names, prices, and publish status. The agent can read this and make decisions based on it.

For unattended agent runs, there's a more robust approach: store the token as an environment variable and pipe it in:

```bash
printf '%s\n' "$GUMROAD_ACCESS_TOKEN" | gumroad auth login --with-token --json --no-input
```

This means the agent doesn't depend on a pre-existing keyring or config file — it just needs the token injected at startup, which is how I run it in the cron-triggered workflow that drafts this blog.

## The Agent Workflow: Product Management

The CLI's `--json` and `--no-input` flags are the key to agent-driven use. Every command returns structured JSON, and the `--no-input` flag prevents any interactive prompts. Combined with `--jq` for inline extraction:

```bash
gumroad products list --all --json --jq '.products[].name' --no-input
```

The agent can pipeline this: check what products exist, decide what needs updating, and run the mutations. Products are created as drafts by default, so you can add a file, set a price, and only publish when ready:

```bash
gumroad products create --name "Weekly SEO Post Pack" --price 5.00 --json --no-input
gumroad products update <id> --file ./pack.zip --json --no-input
gumroad products publish <id> --json --no-input
```

Three commands, no browser, no clicking. The whole flow fits in a shell script or a single agent turn.

## The Gotcha: File Uploads Fail Silently

The first time I tried to attach a file programmatically, I ran this:

```bash
gumroad products create --name "Test" --price 5.00 --file ./myfile.zip --json --no-input
```

The response said `success: true`. The product was created. But when I checked the Gumroad dashboard, the file wasn't attached. The CLI had returned success without actually uploading the file.

The issue: `--file` in `products create` and `products update` expects the file to be uploaded via the two-step flow internally — first upload, then attach. If the upload fails (wrong permissions, oversized file, unreadable path), the CLI reports success for the product creation but silently skips the file attachment. The fix is to upload separately and verify:

```bash
# Upload first, capture the file URL
FILE_URL=$(gumroad files upload ./myfile.zip --json --jq '.file_url' --no-input)

# Then attach it to the product
gumroad products update <id> --file "$FILE_URL" --json --no-input
```

This two-step approach is more reliable. The `files upload` command returns a Gumroad-hosted URL, and `products update --file` accepts that URL. If either step fails, the JSON response tells you exactly what went wrong, including a recovery manifest for interrupted multipart uploads.

## The Agent Workflow: Checking Sales

The agent checks sales daily as part of its heartbeat routine:

```bash
gumroad sales list --after 2026-07-19 --json --jq '.sales[] | {email, product_name, price_cents, created_at}' --no-input
```

This returns a clean list of recent sales. The `--after` flag makes it idempotent — the agent can track the last date it checked and only fetch new sales. For summary views:

```bash
gumroad sales summary --from 2026-07-01 --to 2026-07-20 --group-by product --json --no-input
```

Revenue breakdown by product, in JSON. The agent can parse this and include it in a daily report without ever touching the web UI.

## What I Wired Up

The full automated loop I'm running now:

1. **Content creation** — agent writes a blog post (like this one) and saves it to `_drafts/`
2. **Product update** — agent checks if the product needs a new file or description update via `gumroad products update`
3. **Sales check** — agent fetches new sales, logs them, and alerts me if there's been a refund
4. **Draft email** — agent creates a Gumroad email draft announcing new content, then sends me the preview URL
5. **Commit and push** — the draft post goes to GitHub, the product stays synced

The CLI handles all of it. No web UI, no manual file uploads, no browser automation. The agent stays in its lane — running shell commands and parsing JSON — and the shop stays up to date.

## The One Thing I'd Change

The CLI doesn't have a `--watch` or `--daemon` mode for real-time notifications. For that, you need to set up a webhook (the CLI has `gumroad webhooks create --resource sale --url https://...`) or poll the sales endpoint on a cron. I do the latter — the agent checks sales every 6 hours, which is fine for a low-volume shop. If you're doing high-volume sales, you'd want webhooks to a listener endpoint.

---

*I use this setup daily to manage my shop from the same VPS that runs my agent. If you're building a self-hosted AI ops stack and want the configs pre-assembled, the Self-Hosted AI Assistant Ops Kit at <https://brainscratch.gumroad.com/l/snjbhd> includes the full Gumroad automation workflow.*