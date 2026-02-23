---
layout: post
title: "How to Setup GitHub Notifications on Discord with Webhooks"
date: 2026-02-23 00:00:00 +0000
categories: [tutorial]
tags: [github, discord, webhook, notifications, devops, automation]
author: bqst
summary: "Learn how to setup a GitHub Discord webhook to get real-time repository notifications (commits, PRs, issues) directly in your Discord channel."
---

If your team uses Discord to communicate, setting up a **GitHub Discord webhook** is one of the quickest ways to keep everyone in the loop. Instead of checking GitHub manually, you get real-time notifications for commits, pull requests, issues, and more — directly in a Discord channel.

The setup takes less than five minutes. Here's how to do it.

## Prerequisites

Before you start, make sure you have:

- A **Discord server** where you have the "Manage Webhooks" permission (or admin access)
- A **GitHub repository** where you have admin access (to add webhooks)

That's it. No bots to install, no apps to configure.

## Step 1: Create a Discord Webhook

First, you need to create a webhook in the Discord channel where you want to receive GitHub notifications.

1. Open your Discord server and go to the channel you want to use
2. Click on **Edit Channel** (the gear icon next to the channel name)
3. Go to **Integrations** > **Webhooks**
4. Click **New Webhook**
5. Give it a name (e.g. "GitHub") and optionally set a custom avatar
6. Click **Copy Webhook URL**

Save this URL somewhere — you'll need it in the next step.

**Important:** Treat this webhook URL like a password. Anyone with this URL can send messages to your Discord channel. Never commit it to a public repository or share it publicly.

## Step 2: Add the Webhook to GitHub

Now head over to your GitHub repository.

1. Go to **Settings** > **Webhooks** > **Add webhook**
2. In the **Payload URL** field, paste your Discord webhook URL and **append `/github` to the end**

For example, if your Discord webhook URL is:

```
https://discord.com/api/webhooks/123456789/abcdefg
```

Then your payload URL should be:

```
https://discord.com/api/webhooks/123456789/abcdefg/github
```

The `/github` suffix is critical — it tells Discord to format incoming GitHub payloads as nice embed messages instead of raw JSON.

{:start="3"}
3. Set **Content type** to `application/json`
4. Under "Which events would you like to trigger this webhook?", choose either **Send me everything** or **Let me select individual events** (more on that below)
5. Make sure **Active** is checked
6. Click **Add webhook**

## Step 3: Test It

To verify everything is working, the easiest way is to simply **star your repository** on GitHub (or unstar and re-star it). You should see a notification pop up in your Discord channel almost instantly.

Alternatively, push a commit or open an issue — any event you've subscribed to should trigger a notification.

If nothing shows up, jump to the [Troubleshooting](#troubleshooting) section below.

## Filtering Events

Receiving every single GitHub event can get noisy, especially on active repositories. Instead of "Send me everything", you can select specific events when configuring the webhook:

- **Pushes** — get notified on every push to the repo
- **Pull requests** — track when PRs are opened, closed, or merged
- **Issues** — stay on top of new bug reports and feature requests
- **Issue comments** — follow discussions on issues and PRs
- **Releases** — know when a new version is published

Pick only the events that matter to your team. You can always go back to the webhook settings on GitHub and adjust this later.

## Sending to Discord Forum Threads

If you're using Discord forum channels, you can route GitHub notifications to a specific thread by appending the thread ID as a query parameter:

```
https://discord.com/api/webhooks/123456789/abcdefg/github?thread_id=THREAD_ID
```

Replace `THREAD_ID` with the actual ID of the forum thread. To get a thread ID, enable Developer Mode in Discord settings, then right-click the thread and select **Copy ID**.

## Troubleshooting

If your notifications aren't showing up, check these common issues:

- **Missing `/github` suffix** — This is the most common mistake. Without `/github` at the end of the webhook URL, Discord receives raw JSON and doesn't know how to display it. Double-check your payload URL in GitHub webhook settings.
- **Wrong content type** — Make sure the content type is set to `application/json`, not `application/x-www-form-urlencoded`.
- **Permission issues** — Verify that the webhook still exists in your Discord channel settings. If someone deleted it, GitHub will get a 404 response and eventually disable the webhook.
- **Commit descriptions not showing** — Discord's GitHub integration only displays a limited preview. Long commit messages get truncated, and some event types have minimal formatting. This is a Discord limitation.
- **GitHub shows delivery failures** — Go to your repo's **Settings > Webhooks**, click on the webhook, and scroll down to **Recent Deliveries**. You can inspect the request/response to see what went wrong.

## Wrapping Up

Setting up a GitHub Discord webhook takes just a few minutes and gives your team instant visibility into repository activity. It's one of the simplest DevOps automations you can set up — no third-party bots or services required.

If you need more advanced filtering or custom message formatting, consider looking into [GitHub Actions](https://github.com/features/actions) with a Discord webhook action, but for most teams the built-in integration is more than enough.
