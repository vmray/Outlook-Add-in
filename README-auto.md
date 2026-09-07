# VMRay Report Phishing Outlook Add-in — Automated Deployment Guide

**Latest Version:** 1.1.0 - **Release Date: 01/09/2026** — see [Version History](README.md#version-history) in README.md.

This repository contains the necessary components and instructions to deploy the VMRay Report Phishing Add-in for Outlook. This tool allows users to report suspicious emails directly to a VMRay Incident Response (IR) mailbox for automated analysis.

> **This is the automated (script-driven) deployment guide.** It deploys the add-in from `WebAppAuto/` using a single PowerShell script, and the Web App serves its own manifest at `/manifest.xml`.
>
> For the original manual, portal-driven deployment — which uses `WebApp/` and a manifest you edit by hand — see [README.md](README.md). Both versions are maintained in this repository; pick one and stay with it.

---

## Table of Contents
- [Introduction](#introduction)
  - [Microsoft Outlook Add-ins](#microsoft-outlook-add-ins)
  - [About VMRay](#about-vmray)
- [Prerequisites](#prerequisites)
- [Deployment Overview](#deployment-overview)
- [Quick Start — Cloud Shell](#quick-start--cloud-shell)
  - [Step 1 — Open Cloud Shell](#step-1--open-cloud-shell)
  - [Step 2 — Upload the deployment script](#step-2--upload-the-deployment-script)
  - [Step 3 — Run the deployment script](#step-3--run-the-deployment-script)
  - [Step 4 — Note the manifest URL](#step-4--note-the-manifest-url)
  - [Step 5 — Download the manifest](#step-5--download-the-manifest)
  - [Step 6 — Upload the manifest in Microsoft 365 Admin Center](#step-6--upload-the-manifest-in-microsoft-365-admin-center)
  - [Step 7 — Verify](#step-7--verify)
- [Re-deployment / Reusing an Existing App Registration](#re-deployment--reusing-an-existing-app-registration)
- [Verification](#verification)
  - [Quick health check](#quick-health-check)
  - [End-to-end test](#end-to-end-test)
- [Troubleshooting](#troubleshooting)
- [Summary — Comparison with the Original (Manual) Flow](#summary--comparison-with-the-original-manual-flow)

## Introduction

### Microsoft Outlook Add-ins

Microsoft Outlook Add-ins are web-based extensions that integrate directly into Microsoft Outlook (Desktop, Web, and Mobile). They allow organizations to extend Outlook's functionality by embedding custom workflows directly inside the mailbox experience.

Using Microsoft 365 Single Sign-On (SSO) and Microsoft Graph API, add-ins can securely interact with user mailbox data without storing credentials, while maintaining enterprise-grade security and compliance.

### About VMRay

VMRay is a leading provider of automated malware analysis and advanced threat detection solutions. Using hypervisor-based sandboxing technology, VMRay delivers deep visibility into sophisticated and evasive cyber threats.

The **VMRay Report Phishing Outlook Add-in** enables users to:

- Report suspicious emails with a single click
- Securely forward the original email (including attachments) to a designated VMRay Incident Response (IR) mailbox
- Automatically move reported emails to a dedicated Outlook folder
- Authenticate seamlessly using Microsoft 365 SSO

This add-in streamlines phishing reporting workflows while maintaining security, transparency, and user simplicity.

---

## Prerequisites

| Requirement | Why |
|---|---|
| **Azure Subscription** | To host the Web App that powers the add-in |
| **Global Administrator** (in your Microsoft 365 tenant) | Required to grant tenant-wide consent during deployment and to upload the manifest to the Microsoft 365 Admin Center |
| **VMRay IR Mailbox** | Designated email address (ending in `ir-mailbox.vmray.com`) that receives reported phishing emails. Provisioned via your VMRay Cloud Portal under **Analysis Settings → IR Mailbox** |
| **PowerShell environment** | Azure Cloud Shell |

---

## Deployment Overview

The entire deployment is driven by a single PowerShell script that handles three phases:

| Phase | What it does | Manual? |
|---|---|---|
| **1. App Registration** | Creates the Azure AD App Registration, adds the required Microsoft Graph permissions, mints a client secret | Automated |
| **1.5. Admin consent** | A user authorized to grant tenant-wide consent clicks a one-time URL. The script can either verify consent immediately, or defer it (forward the URL to an admin later) and continue with the rest of the deployment. | **Manual click** (can be deferred) |
| **2. Web App deployment** | Provisions the Azure Web App via an ARM template, configures environment variables | Automated |
| **3. App Reg configuration** | Wires the App Registration to the deployed Web App's domain (Application ID URI, scope, redirect URIs, pre-authorization) | Automated |
| **4. Manifest download** | Customer downloads a ready-to-upload `manifest.xml` from the deployed Web App | One browser click |
| **5. M365 Admin Center upload** | Global Administrator uploads the manifest in M365 Admin Center to deploy to users | **Manual upload** |

**Total customer-side effort: one PowerShell command + one admin-consent click + one manifest upload.**
Phases 2 and 3 can run even if the script operator isn't authorized to grant consent themselves — they can choose **Skip** at the consent step and forward the URL to an admin afterwards. The add-in won't work for end-users until that click happens.

---

## Quick Start — Cloud Shell
### Step 1 — Open Cloud Shell

1. Sign in to [Azure Portal](https://portal.azure.com) with your tenant administrator account
2. Click the **`>_`** Cloud Shell icon in the top-right toolbar
3. Choose **PowerShell** if prompted

### Step 2 — Upload the deployment script

Click **Manage files → Upload** in the Cloud Shell toolbar and upload this file from the repo:

- `WebAppAuto/Scripts/Deploy-VMRayOutlookAddin.ps1`

The ARM template (`azuredeploy.json`) is fetched directly from GitHub by the script — no need to upload it.

### Step 3 — Run the deployment script

```powershell
./Deploy-VMRayOutlookAddin.ps1
```

The script is fully interactive — it'll prompt you for everything it needs. Default values appear in brackets; press Enter to accept.

> **Offline / local override:** If you've made local changes to `azuredeploy.json` or your environment can't reach GitHub, upload the JSON alongside the script and run:
> ```powershell
> ./Deploy-VMRayOutlookAddin.ps1 -TemplateFile ~/azuredeploy.json
> ```
> The local file takes precedence over the default GitHub URL.

You'll be asked:

| Prompt | What to enter |
|---|---|
| Confirm tenant + subscription | If only one subscription is accessible, press Enter to confirm. If multiple are accessible, the script offers a picker — choose 1 to use the current one, or 2 to pick a different one from the list. |
| Create new or use existing App Registration? | Choose 1 (new) for a first-time deployment |
| Display name for the new App Registration | Press Enter for default (`VMRay-Outlook-Addin-App`) |
| **Open the printed consent URL → sign in as someone authorized to grant tenant-wide consent → click Accept** | (Manual step) |
| Consent step: `[1] Access granted - continue`  or  `[2] Skip - to be done manually later` | Choose **1** if consent was just granted (script verifies and waits up to 90s). Choose **2** if you're not authorized to grant it yourself — the script will continue without verifying, and the consent URL will be reprinted at the end for you to forward to an admin. |
| Web App name (1-20 chars, alphanumeric+hyphens) | A globally unique name, e.g., `vmray-outlook-acme`. The script checks the name against Azure before deploying and re-prompts if it's already taken. |
| Resource group name | Existing RG, or a new name (created if missing) |
| Azure region | **Only asked if the resource group is new.** If you reused an existing RG, the script auto-uses that RG's location. For new RGs, press Enter for the default (Central US) or type a closer region (e.g., `Central India`, `North Europe`). |
| App Service SKU | Choose from the numbered list (B1/B2/B3/S1/S2/S3/P1v3/P2v3); press Enter for the default (S1) |
| IR mailbox email address | Your VMRay IR mailbox, e.g., `xxx@us.ir-mailbox.vmray.com` |
| Move reported emails to a folder? | Press Enter for default (No) |
| Proceed with deployment? | Press Enter to confirm |

The deployment runs for 5-7 minutes. The script handles everything automatically including App Registration configuration after the Web App is deployed.

### Step 4 — Note the manifest URL

When the script finishes, it prints a summary like:

```
Web App URL    : https://vmray-outlook-acme.azurewebsites.net
App Client ID  : 13b429ba-f816-4453-9a62-e9a30ee930f2
Tenant ID      : c91fb25b-dd00-4300-bcfa-9dfd39d1451a

Manifest URL (download for M365 Admin Center):
  https://vmray-outlook-acme.azurewebsites.net/manifest.xml

Diagnostics URL (verify deployment health):
  https://vmray-outlook-acme.azurewebsites.net/diagnostics
```

### Step 5 — Download the manifest

Open the Manifest URL from Step 4 in a browser. It downloads `manifest.xml` automatically. **No editing required** — the file is fully populated with your deployment's values.

### Step 6 — Upload the manifest in Microsoft 365 Admin Center

This step requires Global Administrator access to the Microsoft 365 Admin Center.

1. Sign in to [Microsoft 365 Admin Center](https://admin.microsoft.com)
2. Navigate to **Settings → Integrated apps**
3. Click **Add-ins** tab → **Deploy Add-in**
4. Click **Next** → **Upload custom apps**
5. Choose the `manifest.xml` you downloaded in Step 5
6. Click **Upload**
7. Choose deployment scope:
   - **Everyone** — all users in the tenant
   - **Specific users/groups** — selected users (recommended for staged rollouts)
   - **Just me** — recommended for initial testing
8. Click **Deploy**

Microsoft propagates the add-in to user mailboxes within ~30 minutes (sometimes up to 24 hours).

### Step 7 — Verify

After propagation, users see a **Report Phishing** button in their Outlook ribbon when viewing any email. Click it to forward the message to the VMRay IR mailbox for analysis.

See the [Verification](#verification) section for details.

---


## Re-deployment / Reusing an Existing App Registration

If you already have an App Registration from a previous deployment (e.g., migrating to a different Web App, or sharing one App Reg across dev/staging/prod):

When prompted "How do you want to handle the App Registration?", choose **option 2 — Use an existing App Registration**.

The script will:

1. Look up the App Registration by name or AppId
2. If multiple matches exist, ask you to pick one
3. Ask whether to use your existing client secret or mint a fresh one
4. Verify the required permissions (add any that are missing — preserving any custom permissions you've added)
5. Verify admin consent is already granted (or prompt if not)
6. Continue with Web App deployment as normal

This is also useful for re-running the script after a deployment failure — it's idempotent at every step. Re-running into the same resource group is safe: the script clears the stale `waitForWebApp` resource that would otherwise make a second deployment fail (see [Re-deploying into a resource group fails on `waitForWebApp`](#re-deploying-into-a-resource-group-fails-on-waitforwebapp)).

---

## Verification

### Quick health check

Open your Web App's diagnostics URL in a browser:

```
https://YOUR-WEBAPP.azurewebsites.net/diagnostics
```

Expected response:

```json
{
  "status": "ok",
  "timestamp": "...",
  "checks": {
    "env": { "status": "ok", "details": { ... } },
    "recipient": { "status": "ok" },
    "manifestTemplate": { "status": "ok", "path": "..." },
    "moveReportedEmailsToFolder": false
  },
  "manifestUrl": "https://YOUR-WEBAPP.azurewebsites.net/manifest.xml"
}
```

`"status": "ok"` confirms all required env vars are set, the recipient mailbox domain is valid, and the manifest template is in place.

### End-to-end test

1. Sign in to [Outlook on the web](https://outlook.office.com) as a user who has the add-in deployed
2. Open any email
3. Click **Report Phishing** in the ribbon
4. Click **Confirm** in the task pane
5. Wait for the success message: ✅ Email successfully reported to Security Team
6. Open your VMRay portal (e.g., `https://us.cloud.vmray.com`) → **Submissions**
7. Filter: `Interface Type == IR Mailbox`
8. Within 1-2 minutes, your reported email should appear as a new submission

If all of the above succeed, your deployment is fully working.

---

## Troubleshooting

### Consent step — when to choose Skip

When the consent URL is displayed, the script asks how to proceed:

```
Consent step:
  [1] Access granted - continue (verifies and waits up to 90s)
  [2] Skip - to be done manually later by an authorized admin
```

**Choose [1]** when you (or someone working with you in real time) just clicked Accept. The script then verifies the grant via Microsoft Graph and continues automatically once it sees all four permissions consented.

**Choose [2]** when the script operator isn't authorized to grant tenant-wide consent. The script skips verification, runs Phases 2 and 3 (which don't need consent), and reprints the consent URL in the final summary so it can be forwarded to an admin afterwards. The add-in will not work for end-users until that click happens.

### "Consent verification didn't see all permissions granted"

**What's happening:** You chose option [1] above, but after clicking Accept, Microsoft's grant database takes 10-90 seconds to propagate to the Graph API that the script uses to verify consent. The script retries for up to 90 seconds before showing this warning.

**What to do:** The script will offer five options:

1. **Wait another 90s and re-check** — pure propagation lag; re-polls Microsoft Graph without needing another Accept click. Try this first, since the grant often just needs more time to propagate.
2. **Re-open the consent URL and try again** — for when the Accept click may not have registered (browser caching, multiple accounts signed in, etc.). **Tip:** open the URL in an InPrivate/Incognito window for the cleanest session.
3. **I've confirmed consent IS granted in the Portal — continue as granted** — use this if you can already see green checkmarks in Entra ID → App Registration → API permissions but Graph's read-side hasn't caught up. The script trusts your confirmation and proceeds as granted (not deferred), so the final summary won't flag consent.
4. **Continue with deployment anyway** — script proceeds; consent must be granted manually later before users can use the add-in (same as choosing Skip up-front).
5. **Abort** — bail out completely.

**Recommended:** start with option 1 (wait) or 2 (re-click). If you can already see the grant in the Portal (green checkmarks in the **Status** column), use option 3.

### "Temporary server issue" when users click Report Phishing

**Most likely cause:** Admin consent wasn't actually granted, even though the script may have continued.

**To diagnose:** Open the Web App's Log stream in Azure Portal. Look for:

```
forwardMail ERROR: Failed to obtain Graph access token
```

This is the signature of missing consent.

**To fix:** Open the consent URL again as a Global Admin:

```
https://login.microsoftonline.com/YOUR-TENANT-ID/adminconsent?client_id=YOUR-APP-CLIENT-ID
```

Click Accept. Wait 1-2 minutes for propagation. Have the user retry.

### Script can't find the ARM template (when using -TemplateFile)

**Symptom:** `ARM template not found at: <path>`

**Cause:** You passed `-TemplateFile` pointing at a path that doesn't exist (typo, wrong directory, or the file wasn't uploaded to Cloud Shell).

**Fix — Option A (recommended):** drop the `-TemplateFile` flag entirely. The script fetches the template from GitHub by default — no upload needed.

```powershell
./Deploy-VMRayOutlookAddin.ps1
```

**Fix — Option B:** if you really want a local file (offline / customized), verify the path:

```powershell
ls ~/azuredeploy.json
./Deploy-VMRayOutlookAddin.ps1 -TemplateFile ~/azuredeploy.json
```

### Multiple App Registrations with the same name

**Symptom:** The script finds multiple App Registrations matching your input and asks you to pick one.

**Cause:** Previous test deployments left duplicates in your tenant. Azure AD allows multiple App Regs with the same display name.

**Fix:** The script disambiguates by showing a numbered list with Creation dates and AppIds. Pick the one you want. Cleanup of unused ones can be done manually in Entra ID → App registrations after deployment.

### "Web App name is not available"

**What's happening:** The Web App name becomes a public hostname (`<name>.azurewebsites.net`), so it has to be unique across all of Azure — not just your subscription. Before starting the deployment, the script asks Azure's name-availability API whether your chosen name is free.

**What the script does:** It catches the clash in a couple of seconds and re-prompts for a different name, instead of letting a multi-minute deployment fail late with a "Website with given name already exists (Conflict)" error. It loops until you give it an available name, so a taken name never aborts the run. This also applies to a name passed in via `-WebAppName`.

**What to do:** Enter a different, more specific name — e.g., `vmray-outlook-acme` rather than `vmray-outlook`.

**If you see "Could not verify name availability - continuing" instead:** the availability check itself couldn't run (a transient API error, or your account lacks permission to call it). This is only a warning — the script proceeds, and any real conflict surfaces during deployment.

### Re-deploying into a resource group fails on `waitForWebApp`

**What's happening:** The ARM template uses a `Microsoft.Resources/deploymentScripts` resource named `waitForWebApp`. That resource keeps a fixed name and lingers in the resource group after a deployment, with its backing container instance and storage account torn down asynchronously. A second deployment into the same resource group collides with those leftovers and fails.

**What the script does:** Before deploying, it looks for a stale `waitForWebApp` resource in the target resource group and deletes it, forcing a full teardown so the redeploy is clean. You'll see `Removing stale 'waitForWebApp' deployment script from a previous run...` when this happens — that message is expected on any re-run, not a problem.

**If you're deploying the ARM template by hand** (not through the script), delete the `waitForWebApp` resource from the resource group first, or deploy into a fresh resource group.

### Manifest URL shows "Error" or "Application Error" on first click

**What's happening:** After the deployment finishes, the Node.js runtime inside the App Service container has only just started. The first HTTP request arrives while it's still booting — typically 10-30 seconds — so Azure's frontend serves a generic error page instead of the manifest.

**What the script does:** It polls the `/diagnostics` endpoint after Phase 3 and only prints the final summary once the Web App answers with HTTP 200, retrying every 5 seconds for up to 60 seconds. In the vast majority of cases you'll never see this error.

**If you still see it:** wait about 30 seconds and refresh, or paste the URL into a new browser tab. The URL itself is correct — this is purely a cold-start timing issue. If the warm-up poll times out, the script prints a yellow `Web App didn't respond after 60s` warning; give it another minute before the URLs become reachable.

---

## Summary — Comparison with the Original (Manual) Flow

| Step | Original (6 phases) | New (script-driven) |
|---|---|---|
| App Registration setup | ~15 portal clicks | 1 prompt (`./Deploy-VMRayOutlookAddin.ps1`) |
| Admin consent | Manual click | Manual click (unavoidable) |
| Web App deployment | Portal "Deploy to Azure" + manual env var | Auto via script |
| App Reg configuration | ~10 portal clicks | Auto via script |
| Manifest preparation | Manual find/replace in `manifest.xml` | Auto via `/manifest.xml` route |
| M365 Admin Center upload | 1 admin action | 1 admin action (unavoidable) |
| **Total manual interactions** | ~30 portal clicks across multiple pages | **3 actions** (1 PowerShell command + 1 consent click + 1 manifest upload) |

The single-command deployment is the recommended path for all new installations. The legacy step-by-step guide in the original `README.md` remains available for documentation purposes.
