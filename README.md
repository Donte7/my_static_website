# Static Site CI/CD Pipeline Lab
### Guided by Mr. Robot — DevOps & Cloud Security Engineer

---

## What We Built

A fully automated CI/CD pipeline that deploys a static website to **Azure Blob Storage** using **GitHub Actions** — triggered automatically every time you push code from **VS Code**.

Every manual step from the SOP is now automated. You write code, push it, and the pipeline handles the rest.

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
| Azure CLI | Deploys files to $web container |
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

Before running this pipeline you need:

- [ ] An active Microsoft Azure account
- [ ] An Azure Storage Account with static website hosting enabled
- [ ] A GitHub account and repository
- [ ] VS Code installed on your Mac
- [ ] Git installed on your Mac
- [ ] 4 GitHub Secrets configured (see below)

---

## Phase 1 — Azure Setup (Manual — One Time Only)

These steps follow the SOP and are done once in the Azure Portal.

### Step 1 — Create a Storage Account

1. Log in to **https://portal.azure.com**
2. Search for **Storage accounts** → click **+ Create**
3. Configure:
   - **Resource Group:** Create new → `RG-StaticSite-Lab`
   - **Storage Account Name:** `yourname staticsite` (lowercase, no hyphens, globally unique)
   - **Region:** East US
   - **Performance:** Standard
   - **Redundancy:** Locally-redundant storage (LRS)
4. Click **Review** → **Create**
5. Wait 30–60 seconds → click **Go to resource**

> ⚠️ **COMMON ERROR:** Storage account names must be globally unique across ALL of Azure — 3 to 24 characters, all lowercase, letters and numbers only. No hyphens or underscores.

### Step 2 — Enable Static Website Hosting

1. In your Storage Account left menu → **Data management** → **Static website**
2. Click **Enabled**
3. Set **Index document name:** `index.html` (case-sensitive)
4. Set **Error document path:** `404.html`
5. Click **Save**
6. Copy the **Primary endpoint URL** — save it somewhere safe

> ⚠️ **COMMON ERROR:** If you don't see Static website in the menu, your account was created as Blob Storage instead of StorageV2. Delete and recreate it — the portal defaults to StorageV2 if you follow the steps above.

---

## Phase 2 — VS Code Project Setup

### Step 3 — Open Terminal in VS Code

Press `` Command + ` `` to open the integrated terminal.

You should see:
```
yourname@MacBook ~ %
```

### Step 4 — Create Your Project Folder

```bash
mkdir my-static-site
cd my-static-site
pwd
```

Expected output:
```
/Users/yourname/my-static-site
```

### Step 5 — Open Folder in VS Code

```bash
code .
```

> ⚠️ **COMMON ERROR:** If you get `command not found: code` — open VS Code → `Command + Shift + P` → type `Shell Command` → select **Install 'code' command in PATH**.

### Step 6 — Create Project Files

```bash
touch index.html
touch 404.html
mkdir -p .github/workflows
touch .github/workflows/deploy.yml
ls -la
```

Expected output:
```
.github/
404.html
index.html
```

> ⚠️ **COMMON ERROR:** `.github` is a hidden folder. Always use `ls -la` (not just `ls`) to see hidden files and folders on Mac.

### Step 7 — Add Content to index.html

Click `index.html` in VS Code sidebar and paste:

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

Save with `Command + S`.

### Step 8 — Add Content to 404.html

Click `404.html` in VS Code sidebar and paste:

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

Save with `Command + S`.

---

## Phase 3 — The CI/CD Pipeline File

### Step 9 — Add deploy.yml

Click `.github/workflows/deploy.yml` in the VS Code sidebar and paste:

```yaml
# ============================================================
# PIPELINE: Deploy Static Site to Azure Blob Storage
# Automates SOP: Static Website on Azure Blob Storage
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
  # Checks your HTML before anything touches Azure
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
  # Uploads files to Azure — only runs if validate passed
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

Save with `Command + S`.

> ⚠️ **COMMON ERROR:** YAML is extremely sensitive to indentation. Every level uses exactly 2 spaces — never tabs. If you see red underlines in VS Code, check your indentation first.

---

## Phase 4 — GitHub Secrets Setup

Your pipeline needs 4 secrets stored in GitHub. These are encrypted — nobody can read them after saving, including you.

### Where to Add Secrets

GitHub repo → **Settings** → **Secrets and variables** → **Actions** → **New repository secret**

### Secret 1 — AZURE_STORAGE_ACCOUNT

| Field | Value |
|---|---|
| Name | `AZURE_STORAGE_ACCOUNT` |
| Value | Your storage account name e.g. `charlesstaticlab` |

**Where to find it:** Azure Portal → Storage accounts → your account name

### Secret 2 — AZURE_STORAGE_KEY

| Field | Value |
|---|---|
| Name | `AZURE_STORAGE_KEY` |
| Value | Your storage account key1 value |

**Where to find it:** Azure Portal → Storage Account → Security + networking → Access keys → key1 → Show → Copy

> 🔴 **SECURITY ALERT:** This key gives full access to your storage account. Never paste it into code, chat, or email. GitHub Secrets is the only place it should live.

### Secret 3 — AZURE_STATIC_ENDPOINT

| Field | Value |
|---|---|
| Name | `AZURE_STATIC_ENDPOINT` |
| Value | Your primary endpoint URL e.g. `https://charlesstaticlab.z13.web.core.windows.net/` |

**Where to find it:** Azure Portal → Storage Account → Data management → Static website → Primary endpoint

### Secret 4 — AZURE_CREDENTIALS (Service Principal JSON)

This is the most important secret. It gives your pipeline permission to log into Azure.

**Step 1 — Open Azure Cloud Shell**

In Azure Portal click the `>_` icon in the top right. Choose **Bash**.

**Step 2 — Get your Subscription ID**

```bash
az account show --query id --output tsv
```

Copy the output.

**Step 3 — Create the Service Principal**

```bash
az ad sp create-for-rbac \
  --name "github-static-site-deployer" \
  --role "Storage Blob Data Contributor" \
  --scopes /subscriptions/YOUR_SUBSCRIPTION_ID/resourceGroups/RG-StaticSite-Lab \
  --sdk-auth
```

**Step 4 — Copy the entire JSON output**

```json
{
  "clientId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "clientSecret": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "subscriptionId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "tenantId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "activeDirectoryEndpointUrl": "https://login.microsoftonline.com",
  "resourceManagerEndpointUrl": "https://management.azure.com/",
  "activeDirectoryGraphResourceId": "https://graph.windows.net/",
  "sqlManagementEndpointUrl": "https://management.core.windows.net:8443/",
  "galleryEndpointUrl": "https://gallery.azure.com/",
  "managementEndpointUrl": "https://management.core.windows.net/"
}
```

Copy from `{` to `}` — the entire block, nothing more, nothing less.

| Field | Value |
|---|---|
| Name | `AZURE_CREDENTIALS` |
| Value | The entire JSON block above |

> ⚠️ **COMMON ERROR:** If a WARNING message appears above the JSON in Cloud Shell — do NOT include it. Copy only the JSON block. Any extra text breaks the secret instantly.

> 🔴 **SECURITY ALERT:** The Service Principal is scoped to `Storage Blob Data Contributor` on your resource group only. This is least privilege — it can only upload files to your storage account, nothing else. Never use Owner or Contributor at the subscription level for a pipeline.

---

## Phase 5 — Git Commands (In Order)

Run these one at a time in your VS Code terminal. Wait for each to finish before running the next.

```bash
# 1. Confirm you are in your project folder
pwd
```
✅ Should show: `/Users/yourname/my-static-site`

```bash
# 2. Initialize Git
git init
```
✅ Should show: `Initialized empty Git repository...`

```bash
# 3. Set your Git name
git config --global user.name "Your Name"
```
✅ No output = success

```bash
# 4. Set your Git email — use your GitHub account email
git config --global user.email "you@example.com"
```
✅ No output = success

```bash
# 5. Check what files Git sees
git status
```
✅ Should show 3 files in red — not staged yet

```bash
# 6. Stage all files
git add .
```
✅ No output = success

```bash
# 7. Confirm files are staged
git status
```
✅ Should show 3 files in green — ready to commit

```bash
# 8. Create your first commit
git commit -m "Initial commit: static site + CI/CD pipeline"
```
✅ Should show: `3 files changed...`

```bash
# 9. Set branch name to main
git branch -M main
```
✅ No output = success

```bash
# 10. Connect to your GitHub repo
git remote add origin https://github.com/yourusername/my-static-site.git
```
✅ No output = success

```bash
# 11. Verify the GitHub connection
git remote -v
```
✅ Should show your GitHub URL twice

```bash
# 12. Push your code to GitHub
git push -u origin main
```
✅ Should show files uploading to GitHub

> ⚠️ **COMMON ERROR:** If Step 12 asks for a password — use your **GitHub Personal Access Token**, NOT your GitHub account password. Go to GitHub → Settings → Developer settings → Personal access tokens → Generate new token → check `repo` scope → copy and use as password.

---

## Phase 6 — Watch Your Pipeline Run

1. Go to your GitHub repo
2. Click the **Actions** tab
3. Click your workflow run **"Deploy Static Site to Azure"**
4. Watch both jobs complete:

```
✅ Validate HTML Files
✅ Deploy to Azure Blob Storage
```

5. Expand **"Verify site is live"** — you should see:
```
HTTP Status: 200
Site is live and healthy
```

6. Open your Primary Endpoint URL in a browser — you should see:
```
Hello from Azure Blob Storage!
```

---

## SOP to Pipeline Mapping

| SOP Phase | What Automates It |
|---|---|
| Phase 1: Create Storage Account | Done once manually in Azure Portal |
| Phase 2: Enable Static Website | Done once manually in Azure Portal |
| Phase 3 Step 7: Create index.html | Lives in your VS Code project |
| Phase 3 Step 8: Upload to $web | `az storage blob upload-batch` step |
| Phase 4 Step 9: Visit your site | `curl` smoke test step |
| Phase 6: Teardown | Still manual — pipelines don't delete infra |

---

## Common Errors Reference

| Error | Cause | Fix |
|---|---|---|
| `command not found: code` | VS Code shell command not installed | `Command + Shift + P` → Install 'code' command in PATH |
| Pipeline login fails with SyntaxError | AZURE_CREDENTIALS JSON is incomplete or has extra text | Regenerate the Service Principal, copy only the `{...}` block |
| `git push` asks for password and rejects it | GitHub no longer accepts account passwords | Use a Personal Access Token instead |
| Pipeline runs both jobs at same time | Missing `needs: validate` in deploy job | Add `needs: validate` under the deploy job |
| Files don't appear on the live site | Uploaded to wrong container | Confirm files are in `$web` not another container |
| `.github` folder not visible | It's a hidden folder | Use `ls -la` to see hidden files |
| 404 on the live site | index.html named incorrectly | Check for `.txt` extension — must be exactly `index.html` |
| Pipeline uploads internal files to Azure | Missing `--pattern "*.html"` flag | Always use `--pattern "*.html"` in upload-batch command |

---

## Security Alerts Summary

| Alert | Rule |
|---|---|
| Credentials in code | NEVER put passwords, keys, or tokens in your code or YAML files |
| Storage Account Key | Treat like a password — GitHub Secrets only |
| Service Principal scope | Use `Storage Blob Data Contributor` scoped to resource group only — never Owner |
| Upload pattern | Always use `--pattern "*.html"` — never upload everything blindly |
| .github folder in $web | Never let pipeline config files reach your public container |
| Information disclosure | Never expose internal file structure, account names, or configs publicly |
| Repo visibility | Keep repo private until you have reviewed all files and configs |

---

## Key Concepts Learned

**CI/CD Pipeline**
A system that automatically builds, tests, and deploys your code every time you push a change. Replaces all manual steps with automation.

**Quality Gate**
A checkpoint that must pass before the next stage runs. In our pipeline, `needs: validate` ensures broken HTML never reaches Azure.

**GitHub Secrets**
An encrypted vault inside your GitHub repo. Credentials go in here — never in your code.

**Service Principal**
A dedicated robot identity in Azure with limited permissions. Used by pipelines to authenticate without using your personal login.

**Least Privilege**
A security principle — give every identity only the minimum permissions it needs to do its job. Our Service Principal can only upload files, nothing else.

**$web Container**
The special Azure Blob Storage container that serves files publicly as a website. Only files in `$web` are accessible via your public URL.

---

## Quick Reference — Git Commands for Every Future Deploy

After your initial setup, every future deploy is just 3 commands:

```bash
git add .
git commit -m "describe what you changed"
git push
```

That's it. The pipeline handles everything else automatically.

---

## Azure Teardown (When You're Done)

To avoid any ongoing charges, delete your resource group:

1. Azure Portal → search **Resource groups**
2. Click **RG-StaticSite-Lab**
3. Click **Delete resource group**
4. Type `RG-StaticSite-Lab` to confirm
5. Click **Delete**

This removes the storage account, $web container, and all files. Your pipeline will fail on future runs until you recreate the infrastructure.

---

