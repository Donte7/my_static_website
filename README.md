# Static Site CI/CD Pipeline — Azure Blob Storage & GitHub Actions
![Deploy](https://github.com/Donte7/ntfs-file-server-lab/actions/workflows/deploy.yml/badge.svg)



A fully automated CI/CD pipeline that deploys a static website to Azure Blob Storage, triggered automatically on every push to GitHub. The pipeline runs HTML validation as a quality gate before deployment, authenticates to Azure using a least-privilege Service Principal stored in GitHub Secrets, uploads files to the `$web` container via Azure CLI, and runs a smoke test to confirm the live site is healthy — turning a manual, multi-step deployment process into a simple `git push`.

**Pipeline stages:**
- **Validate** — HTMLHint checks `index.html` and `404.html` before anything touches Azure
- **Deploy** — Uploads validated files to the Azure `$web` static hosting container
- **Verify** — Smoke test confirms the live endpoint returns HTTP 200

**Stack:** VS Code, Git/GitHub, GitHub Actions, GitHub Secrets, Azure Blob Storage, Azure CLI, HTMLHint

---

## The Full Automated Flow

```
You edit code in VS Code
        ↓
git add .
git commit -m "your message"
git push
        ↓
GitHub receives your code
        ↓
Pipeline triggers AUTOMATICALLY
        ↓
✅ Validate HTML files
✅ Deploy to Azure $web container
✅ Smoke test confirms site is live
        ↓
Your site is updated on Azure 🚀
```

---

## Tech Stack

| Tool | Role |
|---|---|
| VS Code | Code editor and terminal |
| Git | Version control |
| GitHub | Code hosting and pipeline platform |
| GitHub Actions | CI/CD automation engine |
| GitHub Secrets | Encrypted credential storage |
| Azure Blob Storage | Static website hosting |
| Azure CLI | Deploys files to `$web` container |
| HTMLHint | HTML validation / quality gate |

---

## Project Folder Structure

```
my-static-site/
├── .github/
│   └── workflows/
│       └── deploy.yml      ← CI/CD pipeline brain
├── index.html              ← your homepage
└── 404.html                ← your error page
```

---

## Prerequisites

- [ ] An active Microsoft Azure account
- [ ] An Azure Storage Account with static website hosting enabled
- [ ] A GitHub account and repository
- [ ] VS Code installed on your machine
- [ ] Git installed on your machine
- [ ] 4 GitHub Secrets configured (see below)

---

## Phase 1 — Azure Setup (Manual, One Time Only)

### Step 1 — Create a Storage Account

1. Log in to [portal.azure.com](https://portal.azure.com)
2. Search for **Storage accounts** → click **+ Create**
3. Configure:
   - **Resource Group:** Create new → `RG-StaticSite-Lab`
   - **Storage Account Name:** lowercase, no hyphens, globally unique (e.g. `yourname staticsite`)
   - **Region:** East US
   - **Performance:** Standard
   - **Redundancy:** Locally-redundant storage (LRS)
4. Click **Review** → **Create**
5. Wait 30–60 seconds → click **Go to resource**

> ⚠️ **Common error:** Storage account names must be globally unique across all of Azure — 3 to 24 characters, lowercase letters and numbers only, no hyphens or underscores.

### Step 2 — Enable Static Website Hosting

1. In your Storage Account, go to **Data management** → **Static website**
2. Click **Enabled**
3. Set **Index document name:** `index.html` (case-sensitive)
4. Set **Error document path:** `404.html`
5. Click **Save**
6. Copy the **Primary endpoint URL** and save it

> ⚠️ **Common error:** If "Static website" isn't in the menu, the account was created as Blob Storage instead of StorageV2. Delete and recreate it.

---

## Phase 2 — VS Code Project Setup

### Step 3 — Open Terminal in VS Code

Press `Cmd + ` ` (Mac) to open the integrated terminal.

### Step 4 — Create the Project Folder

```bash
mkdir my-static-site
cd my-static-site
pwd
```

### Step 5 — Open the Folder in VS Code

```bash
code .
```

> ⚠️ **Common error:** `command not found: code` → In VS Code, `Cmd + Shift + P` → "Shell Command" → **Install 'code' command in PATH**.

### Step 6 — Create Project Files

```bash
touch index.html
touch 404.html
mkdir -p .github/workflows
touch .github/workflows/deploy.yml
ls -la
```

> ⚠️ **Common error:** `.github` is a hidden folder — use `ls -la` (not `ls`) to confirm it exists.

### Step 7 — Add Content to `index.html`

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>Azure Static Site</title>
</head>
<body>
  <h1>Hello from Azure Blob Storage!</h1>
  <p>This static site is hosted in the cloud.</p>
</body>
</html>
```

### Step 8 — Add Content to `404.html`

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>Page Not Found</title>
</head>
<body>
  <h1>404 - Page Not Found</h1>
  <p>The page you are looking for does not exist.</p>
</body>
</html>
```

---

## Phase 3 — The CI/CD Pipeline File

### Step 9 — Add `deploy.yml`

```yaml
# ============================================================
# PIPELINE: Deploy Static Site to Azure Blob Storage
# Trigger: Every push to main branch
# ============================================================

name: Deploy Static Site to Azure

on:
  push:
    branches:
      - main
  workflow_dispatch:

jobs:

  # ── JOB 1: VALIDATE ──────────────────────────────────────
  validate:
    name: Validate HTML Files
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install HTMLHint
        run: npm install -g htmlhint

      - name: Validate index.html
        run: htmlhint index.html

      - name: Validate 404.html
        run: htmlhint 404.html

  # ── JOB 2: DEPLOY ────────────────────────────────────────
  deploy:
    name: Deploy to Azure Blob Storage
    runs-on: ubuntu-latest
    needs: validate

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Login to Azure
        uses: azure/login@v2
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}

      - name: Upload files to Azure $web container
        uses: azure/CLI@v2
        with:
          inlineScript: |
            az storage blob upload-batch \
              --account-name ${{ secrets.AZURE_STORAGE_ACCOUNT }} \
              --account-key ${{ secrets.AZURE_STORAGE_KEY }} \
              --destination '$web' \
              --source . \
              --pattern "*.html" \
              --overwrite true

      - name: Verify site is live
        run: |
          STATUS=$(curl -o /dev/null -s -w "%{http_code}" \
            ${{ secrets.AZURE_STATIC_ENDPOINT }})
          echo "HTTP Status: $STATUS"
          if [ "$STATUS" != "200" ]; then
            echo "Site check failed — returned $STATUS"
            exit 1
          else
            echo "Site is live and healthy"
          fi

      - name: Print live URL
        run: echo "Deployed to → ${{ secrets.AZURE_STATIC_ENDPOINT }}"
```

> ⚠️ **Common error:** YAML is sensitive to indentation — use exactly 2 spaces per level, never tabs.

---

## Phase 4 — GitHub Secrets Setup

Add these under **Settings → Secrets and variables → Actions → New repository secret**.

| Secret | Value | Where to find it |
|---|---|---|
| `AZURE_STORAGE_ACCOUNT` | Your storage account name | Azure Portal → Storage accounts |
| `AZURE_STORAGE_KEY` | Your storage account key1 | Storage Account → Security + networking → Access keys |
| `AZURE_STATIC_ENDPOINT` | Your primary endpoint URL | Storage Account → Data management → Static website |
| `AZURE_CREDENTIALS` | Service Principal JSON (see below) | Generated via Azure CLI |

> 🔴 **Security alert:** The storage account key gives full access to your account. Never paste it into code, chat, or email — GitHub Secrets only.

### Creating `AZURE_CREDENTIALS`

1. Open **Azure Cloud Shell** (Bash) from the Azure Portal
2. Get your Subscription ID:
   ```bash
   az account show --query id --output tsv
   ```
3. Create the Service Principal:
   ```bash
   az ad sp create-for-rbac \
     --name "github-static-site-deployer" \
     --role "Storage Blob Data Contributor" \
     --scopes /subscriptions/YOUR_SUBSCRIPTION_ID/resourceGroups/RG-StaticSite-Lab \
     --sdk-auth
   ```
4. Copy the entire JSON output (from `{` to `}`, nothing more) and paste it as the value of `AZURE_CREDENTIALS`.

> ⚠️ **Common error:** If a WARNING message appears above the JSON in Cloud Shell, do not include it — only the JSON block.

> 🔴 **Security alert:** This Service Principal is scoped to `Storage Blob Data Contributor` on the resource group only — least privilege. Never use Owner or Contributor at the subscription level for a pipeline.

---

## Phase 5 — Git Commands (First-Time Setup)

```bash
git init
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
git add .
git commit -m "Initial commit: static site + CI/CD pipeline"
git branch -M main
git remote add origin https://github.com/yourusername/my-static-site.git
git push -u origin main
```

> ⚠️ **Common error:** If `git push` asks for a password, use a **GitHub Personal Access Token**, not your account password. Generate one under GitHub → Settings → Developer settings → Personal access tokens (with `repo` scope).

---

## Phase 6 — Watch the Pipeline Run

1. Go to your GitHub repo → **Actions** tab
2. Open the **"Deploy Static Site to Azure"** workflow run
3. Confirm both jobs pass:
   ```
   ✅ Validate HTML Files
   ✅ Deploy to Azure Blob Storage
   ```
4. Expand **"Verify site is live"** — expect `HTTP Status: 200`
5. Open your Primary Endpoint URL in a browser to confirm the site is live

---

## Common Errors Reference

| Error | Cause | Fix |
|---|---|---|
| `command not found: code` | VS Code shell command not installed | Install via Command Palette → Shell Command |
| Pipeline login fails | `AZURE_CREDENTIALS` JSON incomplete or malformed | Regenerate the Service Principal, copy only the `{...}` block |
| `git push` rejects password | GitHub no longer accepts account passwords | Use a Personal Access Token |
| Both jobs run simultaneously | Missing `needs: validate` | Add `needs: validate` under the deploy job |
| Files missing from live site | Uploaded to wrong container | Confirm files are in `$web` |
| `.github` folder not visible | Hidden folder | Use `ls -la` |
| 404 on live site | `index.html` named incorrectly | Confirm exact filename, no extra extension |
| Internal files uploaded to Azure | Missing `--pattern "*.html"` | Always include the pattern flag in upload-batch |

---

## Security Notes

- Never commit passwords, keys, or tokens to code or YAML files
- Treat the storage account key like a password — GitHub Secrets only
- Scope the Service Principal to `Storage Blob Data Contributor` on the resource group, never Owner
- Always use `--pattern "*.html"` on upload — never upload indiscriminately
- Never let `.github` config files reach the public `$web` container
- Keep the repo private until secrets and configs have been reviewed

---

## Key Concepts

**CI/CD Pipeline** — Automatically builds, tests, and deploys code on every push.

**Quality Gate** — A checkpoint (`needs: validate`) that blocks deployment if validation fails.

**GitHub Secrets** — Encrypted credential storage scoped to the repo.

**Service Principal** — A dedicated, limited-permission identity Azure pipelines use to authenticate.

**Least Privilege** — Granting an identity only the minimum permissions it needs.

**`$web` Container** — The special Blob Storage container that serves files as a public website.

---

## Every Future Deploy

```bash
git add .
git commit -m "describe what you changed"
git push
```

The pipeline handles validation, deployment, and verification automatically.

---

## Teardown

To avoid ongoing charges, delete the resource group:

1. Azure Portal → **Resource groups**
2. Select **RG-StaticSite-Lab**
3. **Delete resource group** → type the name to confirm → **Delete**

This removes the storage account, `$web` container, and all files. The pipeline will fail on future runs until the infrastructure is recreated.

---
## Link to my website 
https://nerdstaticlab.z13.web.core.windows.net/