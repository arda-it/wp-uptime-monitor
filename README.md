# WP Engine uptime monitor

Checks every production WordPress install on your WP Engine account on a schedule and alerts you by Slack and/or email when a site goes down, comes back, or has an SSL certificate about to expire. Optionally, Claude adds a short "likely cause" to each down alert.

## What it checks

For each site: the homepage (with a cache-busting query string) must return a non-error status and must not contain WordPress failure text such as "Error establishing a database connection"; `/wp-json/` must not return a 5xx; and the SSL certificate must have more than 14 days left. A site is only marked down after 3 failed attempts, 15 seconds apart. If 3 or more sites fail at once, you get one grouped "probably a WP Engine platform issue" alert instead of one per site.

## Setup

1. **WP Engine API credentials.** In the WP Engine User Portal, open API Access (https://my.wpengine.com/apiaccess), enable API access for the account if it isn't already, and generate credentials. Use a user who can see every install you want monitored. Copy the User ID and Password.
2. **Slack webhook.** Go to https://api.slack.com/apps → Create New App → From scratch. Open Incoming Webhooks, switch it on, click "Add New Webhook to Workspace", and pick the alerts channel. Copy the webhook URL.
3. **(Optional) Anthropic API key** for the Claude diagnosis, from https://console.anthropic.com. Without it, alerts still work, just without the "likely cause" line.
4. **GitHub repo.** Create a new repository and upload `monitor.py`, `README.md`, and `.github/workflows/uptime.yml`, keeping that folder path exactly.
5. **Secrets.** In the repo: Settings → Secrets and variables → Actions → New repository secret. Add `WPE_API_USER_ID`, `WPE_API_PASSWORD`, `SLACK_WEBHOOK_URL`, and optionally `ANTHROPIC_API_KEY`.
6. **Test.** Actions tab → "WordPress uptime monitor" → Run workflow. The log should end with something like `Done: 12/12 up`. To test alerts, temporarily add a dead domain as the `EXTRA_SITES` secret (below) and run again.

Slack and email are both optional; set up either or both.

## Email alerts

Add these as repository secrets:

| Secret | Example |
|---|---|
| `SMTP_HOST` | `smtp.gmail.com` |
| `SMTP_PORT` | `587` (or `465`) |
| `SMTP_USER` | `alerts@youragency.com` |
| `SMTP_PASSWORD` | app password or provider SMTP key |
| `EMAIL_FROM` | `Uptime Monitor <alerts@youragency.com>` (defaults to `SMTP_USER`) |
| `ALERT_EMAILS` | `you@agency.com, colleague@agency.com` — get every alert |
| `SITE_EMAILS` | `clientsite.com=client@client.com pm@agency.com; other.com=owner@other.com` — extra people for specific sites only |

Each recipient gets **one email per run** listing all of their alerts, so an outage across many sites doesn't flood anyone's inbox.

Which SMTP to use:

- **Google Workspace / Gmail:** turn on 2-Step Verification for the sending account, create an App Password (Google Account → Security → App passwords), and use it as `SMTP_PASSWORD`. Host `smtp.gmail.com`, port `587`.
- **Transactional email service** (Postmark, SendGrid, Mailgun, Amazon SES, Resend, etc.): most reliable for automated alerts. Verify your sending domain with them, then use the SMTP host, username and key they give you.
- **Microsoft 365 / Outlook:** Microsoft has been phasing out password-based SMTP sign-in, so a plain mailbox password may be refused. Use a transactional service instead unless your admin confirms SMTP AUTH is allowed.

## Optional settings

Also stored as **secrets** (not variables), so they stay masked in the run logs:

- `EXCLUDE_DOMAINS`: comma-separated domains to skip.
- `EXTRA_SITES`: comma-separated domains to monitor that aren't on WP Engine.

GitHub won't show a secret's value after you save it, so keep your own copy of longer ones like `SITE_EMAILS`.

Other tunables (timeouts, retries, SSL warning days, reminder interval for long outages) are environment variables at the top of `monitor.py`.

## Cost and repo visibility

GitHub bills each scheduled run as at least one minute. Every 5 minutes is about 8,640 runs a month.

- **Public repo:** unlimited free minutes on standard runners, so every 5 minutes costs nothing. No secrets are in the code, and the logs only print counts, not domain names. GitHub disables schedules in public repos after 60 days with no commits; it emails you first, and re-enabling is one click on the Actions tab.
- **Private repo, free plan:** 2,000 minutes a month, so change the cron to `*/30 * * * *`.
- **Private repo, every 5 minutes:** works, but the minutes over your plan's allowance are billed.

GitHub can delay scheduled runs by a few minutes when it is busy, so treat "every 5 minutes" as approximate.

## If the monitor itself breaks

If a run fails (for example, bad credentials), GitHub emails whoever last edited the workflow file. If the WP Engine API is unreachable, the monitor keeps checking the last known list of sites.
