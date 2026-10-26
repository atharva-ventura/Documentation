# Ventura Status Page — UAT Deployment Pipeline

## Problem Statement

The `ventura_status` project had no automated deployment pipeline. Every time code changed, someone had to manually SSH into the UAT server, pull the latest code, install dependencies, set the environment, and restart the app. This was slow, error-prone, and inconsistent with how all other UAT services are deployed.

---

## Root Cause Analysis (RCA)

| # | Issue | Cause |
|---|---|---|
| 1 | No GitHub Actions workflow existed for `ventura_status` | Project was added to the repo but never wired into CI/CD |
| 2 | `test/schedule.test.js` not found on server | The test file referenced in `package.json` (`npm test`) does not exist on the UAT server, causing the pipeline to fail at the test job |
| 3 | `uat-ventura-status.service` not found | No systemd service was ever created on the UAT host for this app, so `systemctl restart` failed with exit code 5 |
| 4 | No process manager configured | Without PM2 or systemd, `npm start` exits as soon as the GitHub Actions job finishes — the app does not stay running |

---

## Solution

### 1. Created GitHub Actions Workflow

Created `.github/workflows/uat-ventura-status.yml` that automatically triggers on every push to `sprint_9` that touches `src/api/ventura_status/**`.

Pipeline jobs in order:

```
Checkout → VenturaStatusInstallPackages → SetEnvVenturaStatus → DeployVenturaStatus → SlackNotification
```

| Job | What it does |
|---|---|
| `Checkout` | Checks out the `sprint_9` branch and verifies the commit |
| `VenturaStatusInstallPackages` | Runs `npm install` inside `src/api/ventura_status/` on the UAT server |
| `SetEnvVenturaStatus` | Writes `ENV=qa` to `.env` so `app.js` picks the `qa` config block |
| `DeployVenturaStatus` | Uses PM2 to start or restart the app with `npm start` |
| `SlackNotification` | Sends success or failure notification to `#github_pr_request` |

### 2. Removed the Test Job

The `npm test` job was removed because `test/schedule.test.js` does not exist on the UAT server. Adding it back requires creating the test file first.

### 3. Used PM2 Instead of systemd

Instead of creating a systemd service file manually on the server, PM2 is used as the process manager because:

- No manual server setup needed
- The workflow handles first-time start and subsequent restarts automatically
- Keeps the app running in the background after the pipeline finishes
- Auto-restarts the app if it crashes

Deploy step logic:
```bash
# If process already exists → restart it
pm2 restart uat-ventura-status

# If first time → create and start it
pm2 start npm --name uat-ventura-status -- start

# Persist across server reboots
pm2 save
```

### 4. Environment Configuration

Matches the exact pattern used by all other UAT services:
```bash
echo "ENV=qa" > /home/ubuntu/actions-runner/_work/project-cash/project-cash/src/api/ventura_status/.env
```

This tells `app.js` to load the `qa` block from `config.js` (port 4000, QA Postgres connection).

---

## Files Changed

| File | Change |
|---|---|
| `.github/workflows/uat-ventura-status.yml` | Created — full UAT deployment pipeline |

---

## How to Deploy

Push any change to `src/api/ventura_status/**` on the `sprint_9` branch. The pipeline runs automatically.

To trigger manually: go to **GitHub → Actions → Deployment - Ventura Status Page (UAT) → Run workflow**.

---

## First-Time Setup (One-Time Only)

If PM2 is not yet running on the UAT server, run once:

```bash
npm install -g pm2
pm2 startup
```

After the first pipeline run, the process will be registered and persist across reboots via `pm2 save`.
