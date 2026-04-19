# SafeShip Demo — Deploy Risk Scoring

A demo project for [SafeShip](https://safeship.dev) — an ML-powered deploy risk engine.

## What's in here

| File | Purpose |
|---|---|
| `index.html` | Single-file React frontend (no npm needed — uses CDN) |
| `Jenkinsfile` | Full CI/CD pipeline with SafeShip risk scoring stage |

## How it works

Every `git push` triggers Jenkins via a GitHub webhook.  
Jenkins runs the pipeline, computes your diff size, and calls the **SafeShip scoring API** before deploying:

```
Git Push → GitHub Webhook → Jenkins → SafeShip Score → Deploy or Block
```

## SafeShip Verdicts

| Score | Verdict | Action |
|---|---|---|
| 0–39 | ✅ SAFE | Deploys normally |
| 40–69 | ⚠️ CAUTION | Continues with warning |
| 70–100 | 🚫 BLOCKED | Pipeline fails — deploy stopped |

## Quick Start

1. Open `index.html` in your browser — no server needed
2. Configure `Jenkinsfile` with your EC2 IP, tenant ID, and API key
3. Push to GitHub, set up the webhook → every commit gets scored automatically

## Jenkins Setup

Update the env vars at the top of `Jenkinsfile`:

```groovy
SAFESHIP_EC2_IP    = 'YOUR-EC2-IP'
SAFESHIP_TENANT_ID = 'ce347a6873784a9b'
SAFESHIP_API_KEY   = '6832057ec6fe4346861b2e2c16dd443e'
```
