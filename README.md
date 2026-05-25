# n8n Automation Platform — Render Deployment

Self-hosted [n8n](https://n8n.io) on Render for managing invoice-chasing and other automation workflows across multiple clients.

---

## What's in this repo

| File | What it does |
|---|---|
| `Dockerfile` | Tells Render which version of n8n to run. You only ever need to edit this to upgrade n8n. |
| `render.yaml` | The full blueprint for your Render infrastructure — web service + Postgres database. Render reads this automatically when you connect the repo. |
| `docker-compose.yml` | Only used if you want to run n8n locally on your own computer for testing. Render ignores this file. |
| `.env.example` | A reference list of every environment variable the app needs. You copy the values into the Render dashboard — do not put real secrets in this file. |

---

## Before you start — what this will cost

Render's free tier is not suitable for a live automation platform because the server goes to sleep after 15 minutes of no traffic, which causes webhooks and scheduled workflows to fail silently.

| Resource | Plan | Monthly cost |
|---|---|---|
| Web service (n8n itself) | Starter | $7 |
| Postgres database | Starter | $7 |
| Persistent disk (for file storage) | 1 GB | ~$0.25 |
| **Total** | | **~$14.25/month** |

This is the minimum for a stable, always-on setup. You can cancel any time.

---

## Step 1 — Generate your encryption key

n8n encrypts every credential you save (Xero tokens, Gmail passwords, etc.) using a secret key. You must generate this **before** you deploy and keep a copy somewhere safe — if you lose it or change it later, all your saved credentials will stop working.

Run this command in your terminal (Mac/Linux) or use an online generator:

```
openssl rand -hex 32
```

Copy the output. It will look something like:

```
a3f82c1d9e7b4056f2a1c8d3e0b5f9a2c7e4d1b8f6a3c0e9d2b5a8f1c4e7b0
```

Save it in your password manager right now. You will paste it into Render in Step 3.

---

## Step 2 — Connect this repo to Render

1. Log in to your [Render dashboard](https://dashboard.render.com).
2. Click **New** → **Blueprint**.
3. Connect your GitHub account if you haven't already, then select this repository.
4. Render will find the `render.yaml` file automatically and show you a summary of what it will create: one web service and one Postgres database.
5. Click **Apply** to start the setup. Render will ask you to fill in environment variables — see Step 3.

---

## Step 3 — Set your environment variables in Render

Before or immediately after clicking Apply, you need to set three variables. You can do this in the Render dashboard under your web service → **Environment**.

### `N8N_ENCRYPTION_KEY` (critical)
Paste the key you generated in Step 1. Do not share this with anyone.

### `N8N_HOST`
This is the domain name Render gives your service. It will look like `n8n-automation.onrender.com`. You will know this after the service is created — go back and fill it in if you need to do it after the first deploy.

### `WEBHOOK_URL`
The full URL including `https://`. Example:
```
https://n8n-automation.onrender.com
```

All other variables in the `render.yaml` are either set automatically or have sensible defaults. You do not need to change them unless you have a specific reason to.

---

## Step 4 — First login and account setup

Once Render finishes deploying (takes 2–5 minutes), open your n8n URL in a browser.

n8n will walk you through a one-time setup:
- Enter your email address and choose a strong password.
- This creates your owner account — you are the only person who can log in.

**Important:** Write this email and password down. If you forget your password there is no "forgot password" email option on a self-hosted install — you would need to reset the database.

---

## Step 5 — Connect your client accounts

Once you are logged in, you can start connecting client accounts:

### Connecting a Xero account
1. In n8n, go to **Credentials** → **New Credential** → search for **Xero OAuth2**.
2. You will need to create a Xero app at [developer.xero.com](https://developer.xero.com) to get a Client ID and Secret.
3. Each client gets their own credential entry with their own Xero login.

### Connecting QuickBooks Online
1. Go to **Credentials** → **New Credential** → search for **QuickBooks OAuth2**.
2. You will need a QuickBooks developer app at [developer.intuit.com](https://developer.intuit.com).

### Connecting Gmail
1. Go to **Credentials** → **New Credential** → **Google OAuth2**.
2. You will need to set up a Google Cloud project and OAuth consent screen — n8n's documentation covers this in detail.

### Connecting Outlook / Microsoft 365
1. Go to **Credentials** → **New Credential** → **Microsoft OAuth2**.
2. Requires an Azure app registration — again, n8n's docs walk through this.

All credentials are stored encrypted in your Postgres database using your `N8N_ENCRYPTION_KEY`.

---

## Managing multiple clients from one dashboard

n8n lets you create separate credentials per client and use them in individual workflows. A typical setup for invoice chasing looks like this:

- **One workflow per client** — each workflow uses that client's Xero/QuickBooks credential and that client's Gmail/Outlook credential.
- Use n8n **tags** to organise workflows by client name.
- Use **scheduled triggers** (cron) to run checks daily or weekly.

There is no built-in client isolation — all your workflows live in the same n8n instance, visible only to you when you log in.

---

## Upgrading n8n

To update to a newer version of n8n:

1. Open `Dockerfile` in this repo.
2. Change the version number on line 1, for example:
   ```
   FROM n8nio/n8n:1.91.3
   ```
   to:
   ```
   FROM n8nio/n8n:1.95.0
   ```
3. Commit and push to GitHub. Render will automatically rebuild and redeploy.

Check for new versions at [github.com/n8n-io/n8n/releases](https://github.com/n8n-io/n8n/releases). Always read the release notes before upgrading — n8n occasionally makes breaking changes.

---

## Running locally for testing (optional)

If you want to test workflows on your own computer before deploying to Render, you need Docker Desktop installed, then run:

```
docker compose up
```

Open [http://localhost:5678](http://localhost:5678) in your browser. This is completely separate from your live Render instance — changes made locally do not affect production.

---

## GDPR note for UK clients

Your n8n instance on Render will run in the US (Oregon) by default. If you are storing or processing personal data from UK/EU clients (which invoice data may count as), you might prefer to host in the EU.

To switch to Frankfurt (EU), add this to both the `services` and `databases` sections of `render.yaml`:

```yaml
region: frankfurt
```

Note: Render's EU region costs the same. You would need to delete and recreate the service to move an existing deployment.

---

## Troubleshooting

**Workflows are not running on schedule**
The web service might be on the free plan and sleeping. Make sure you are on the Starter plan in Render.

**"Invalid encryption key" or credentials not working**
The `N8N_ENCRYPTION_KEY` in Render does not match what was used when credentials were originally saved. Do not change this key after first use.

**Webhooks from Xero/QuickBooks are not arriving**
Make sure `WEBHOOK_URL` is set correctly to your full `https://` Render URL. Webhooks will silently fail if this is wrong.

**n8n shows a blank page or fails to load**
Check the logs in the Render dashboard (your service → Logs tab). Common causes: database not yet ready, missing environment variable, or an in-progress deploy.
