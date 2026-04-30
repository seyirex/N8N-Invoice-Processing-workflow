# N8N Invoice Processing Workflow

Automatically processes invoices (PDF or image) uploaded to Google Drive or via a web form. Powered by Google Gemini AI, results are saved to Google Sheets and a notification email is sent via Gmail. An AI chat agent lets your team query invoice data in plain English.

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Prerequisites](#prerequisites)
3. [Installing Docker (Mac & Windows)](#installing-docker)
4. [Step 1 — Install & Run n8n with Docker](#step-1--install--run-n8n-with-docker)
5. [Step 2 — Get a Google Gemini API Key](#step-2--get-a-google-gemini-api-key)
6. [Step 3 — Set Up Google OAuth Credentials](#step-3--set-up-google-oauth-credentials)
7. [Step 4 — Create the Google Sheet](#step-4--create-the-google-sheet)
8. [Step 5 — Create a Google Drive Invoice Folder](#step-5--create-a-google-drive-invoice-folder)
9. [Step 6 — Import the Workflows into n8n](#step-6--import-the-workflows-into-n8n)
10. [Step 7 — Configure Credentials in n8n](#step-7--configure-credentials-in-n8n)
11. [Step 8 — Update Workflow Placeholders](#step-8--update-workflow-placeholders)
12. [Step 9 — Activate & Test](#step-9--activate--test)
13. [Workflow Reference](#workflow-reference)
14. [Troubleshooting](#troubleshooting)

---

## Architecture Overview

```
Google Drive (new invoice) ─┐
                            ├─▶ Filter (PDF / Image) ─▶ Gemini AI ─▶ Parse & Validate ─▶ Google Sheets ─▶ Gmail notify
Web Form Upload ────────────┘

n8n Chat UI ──▶ Invoice Query Agent ──▶ Gemini AI ──▶ Invoice Data Reader ──▶ Google Sheets
```

**Three workflows are included:**

| File | Purpose |
|------|---------|
| `Invoice Processing — Google Drive to Sheets.json` | Main pipeline — watches Drive, extracts invoice data with Gemini, saves to Sheets, emails you |
| `Invoice Data Reader.json` | Sub-workflow called by the AI agent to fetch all invoice rows from Sheets |
| `Invoice Query Agent.json` | Chat agent — answers natural-language questions about your invoice data |

---

## Prerequisites

| Tool | Version | Notes |
|------|---------|-------|
| Docker | Any recent version | See installation steps below |
| Google Account | — | For Drive, Sheets, Gmail, and Gemini |
| Google Cloud Project | — | Free tier is fine for getting started |

---

## Installing Docker

### macOS

**Option A — Docker Desktop (recommended, easiest)**

1. Go to [https://www.docker.com/products/docker-desktop/](https://www.docker.com/products/docker-desktop/).
2. Click **Download for Mac** — choose **Apple Silicon** (M1/M2/M3/M4) or **Intel chip** based on your Mac.
3. Open the downloaded `.dmg` file and drag **Docker** into your Applications folder.
4. Launch **Docker** from Applications. It will ask for your password to install helper tools — approve it.
5. Wait for the Docker whale icon in the menu bar to show a steady state (not animating).
6. Verify in a terminal:
   ```bash
   docker --version
   ```

**Option B — Homebrew (for developers who already use Homebrew)**

```bash
brew install --cask docker
open /Applications/Docker.app
```

> Docker Desktop requires macOS 12 Monterey or later. For older macOS versions, use [Colima](https://github.com/abiosoft/colima) as an alternative.

---

### Windows

**Option A — Docker Desktop (recommended)**

1. Go to [https://www.docker.com/products/docker-desktop/](https://www.docker.com/products/docker-desktop/).
2. Click **Download for Windows**.
3. Run the installer (`Docker Desktop Installer.exe`).
4. When prompted, make sure **Use WSL 2 instead of Hyper-V** is checked (WSL 2 is faster and required on Windows Home).
5. Follow the installer prompts and restart your computer when asked.
6. After reboot, Docker Desktop launches automatically. Accept the service agreement.
7. Verify in PowerShell or Command Prompt:
   ```powershell
   docker --version
   ```

**WSL 2 setup (required for Windows Home, recommended for all)**

If you do not have WSL 2 installed yet, run this in PowerShell as Administrator **before** installing Docker:

```powershell
wsl --install
```

Restart your computer, then install Docker Desktop.

> Docker Desktop requires Windows 10 64-bit (version 1903 or later) or Windows 11.  
> For Windows Home you **must** use WSL 2 — Hyper-V is not available on Home editions.

---

## Step 1 — Install & Run n8n with Docker

### 1.1 Clone this repository

```bash
git clone https://github.com/seyirex/N8N-Invoice-Processing-workflow
cd N8N-Invoice-Processing-workflow
```

### 1.2 Create your environment file

```bash
cp .env.example .env
```

Open `.env` and set at minimum:

```dotenv
N8N_ENCRYPTION_KEY=<generate with: openssl rand -hex 16>
GENERIC_TIMEZONE=America/New_York   # your timezone
N8N_PUSH_BACKEND=websocket
N8N_DEFAULT_BINARY_DATA_MODE=default  # required for image invoice processing
```

> **Important:** `N8N_DEFAULT_BINARY_DATA_MODE=default` keeps uploaded files in memory so the Gemini image node can read them. Do not remove it.

### 1.3 Start n8n

**Option A — Docker Compose (recommended)**

A `docker-compose.yml` is included. This is the easiest way to manage the container:

```bash
docker compose up -d
```

Useful commands:

```bash
docker compose logs -f n8n      # stream live logs
docker compose down             # stop and remove the container (data is kept)
docker compose pull && docker compose up -d   # update to the latest n8n image
```

> `docker compose down -v` will also delete the `n8n_data` volume — **this wipes all your workflows and credentials**. Only run it if you want a full reset.

**Option B — Plain Docker run**

```bash
docker run -it --rm --name n8n \
  -p 5678:5678 \
  -v n8n_data:/home/node/.n8n \
  --env-file .env \
  docker.n8n.io/n8nio/n8n
```

See `start-command.md` for a commented version of this command.

n8n will be available at **http://localhost:5678**.

### 1.4 Create your n8n owner account

On first launch, n8n walks you through creating an owner account. Complete that before continuing.

---

## Step 2 — Get a Google Gemini API Key

The workflow calls the Gemini API directly via HTTP. You need an API key.

1. Go to [Google AI Studio](https://aistudio.google.com/app/apikey).
2. Click **Create API key** → choose an existing Google Cloud project (or create a new one).
3. Copy the key — you will use it in [Step 7](#step-7--configure-credentials-in-n8n).

> **Free tier:** Gemini 2.5 Flash has a generous free quota. No billing required to get started.

---

## Step 3 — Set Up Google OAuth Credentials

The workflow uses OAuth 2.0 to access **Google Drive**, **Google Sheets**, and **Gmail** on your behalf.

### 3.1 Create a Google Cloud Project

1. Open [Google Cloud Console](https://console.cloud.google.com/).
2. Click the project selector at the top → **New Project** → name it (e.g. `n8n-invoice`) → **Create**.

### 3.2 Enable the required APIs

In your project, go to **APIs & Services → Library** and enable each of these:

- Google Drive API
- Google Sheets API
- Gmail API
- Generative Language API *(for the Invoice Query Agent's native Gemini node)*

### 3.3 Configure the OAuth Consent Screen

1. Go to **APIs & Services → OAuth consent screen**.
2. Choose **External** → **Create**.
3. Fill in **App name** (e.g. `n8n Invoice`) and your email for both support and developer contact fields.
4. Click **Save and Continue** through the remaining screens (no scopes needed here).
5. On the **Test users** screen, add your Google account email → **Save and Continue**.
6. Click **Back to Dashboard**.

### 3.4 Create OAuth 2.0 Client Credentials

1. Go to **APIs & Services → Credentials** → **Create Credentials → OAuth client ID**.
2. Application type: **Web application**.
3. Name: `n8n`.
4. Under **Authorized redirect URIs**, add:
   ```
   http://localhost:5678/rest/oauth2-credential/callback
   ```
   > If running n8n on a server, replace `localhost:5678` with your public domain.
5. Click **Create**. Copy the **Client ID** and **Client Secret**.

---

## Step 4 — Create the Google Sheet

The workflow saves every invoice to a Google Sheet. You must create the sheet with the exact column headers below.

### 4.1 Create the spreadsheet

1. Open [Google Sheets](https://sheets.google.com) → **Blank spreadsheet**.
2. Rename it to **Invoice Database** (or any name you prefer).

### 4.2 Add the column headers

In **Row 1**, type these headers exactly (copy-paste the row below):

| A | B | C | D | E | F | G | H | I | J | K | L | M | N | O | P | Q |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| Invoice Number | Invoice Date | Due Date | Vendor Name | Vendor Address | Vendor Email | Bill To Company | Bill To Address | Line Items | Subtotal | Tax Amount | Total Amount | Currency | Payment Terms | Notes | Source File | Processed At |

### 4.3 Copy the Spreadsheet ID

The spreadsheet ID is in the URL:

```
https://docs.google.com/spreadsheets/d/SPREADSHEET_ID_IS_HERE/edit
```

Copy that ID — you will use it in [Step 8](#step-8--update-workflow-placeholders).

---

## Step 5 — Create a Google Drive Invoice Folder

1. Open [Google Drive](https://drive.google.com).
2. Click **+ New → Folder** → name it **Invoices** (or any name).
3. Open the folder and copy the **folder ID** from the URL:
   ```
   https://drive.google.com/drive/folders/FOLDER_ID_IS_HERE
   ```

Keep this ID handy for [Step 8](#step-8--update-workflow-placeholders).

---

## Step 6 — Import the Workflows into n8n

1. Open n8n at **http://localhost:5678**.
2. Go to **Workflows** (left sidebar) → click **Import from File** (top-right menu ⋮).
3. Import the workflows in this order (order matters because the Agent references the Data Reader's ID):
   1. `Invoice Data Reader.json`
   2. `Invoice Processing — Google Drive to Sheets.json`
   3. `Invoice Query Agent.json`
4. After importing **Invoice Data Reader**, open it and note its workflow ID from the URL:
   ```
   http://localhost:5678/workflow/WORKFLOW_ID_IS_HERE
   ```
   You will need this for the Invoice Query Agent in [Step 8](#step-8--update-workflow-placeholders).

---

## Step 7 — Configure Credentials in n8n

Go to **Settings → Credentials** in n8n and create the following credentials.

### 7.1 Google Drive OAuth2 credential

1. Click **Add Credential → Google Drive OAuth2 API**.
2. Paste your **Client ID** and **Client Secret** from Step 3.4.
3. Click **Connect my account** — a Google sign-in window opens. Approve access.
4. Save. Note the credential name you give it (e.g. `Google Drive account`).

### 7.2 Google Sheets OAuth2 credential

1. Click **Add Credential → Google Sheets OAuth2 API**.
2. Use the **same Client ID and Client Secret**.
3. Connect your account → Save.

### 7.3 Gmail OAuth2 credential

1. Click **Add Credential → Gmail OAuth2 API**.
2. Use the **same Client ID and Client Secret**.
3. Connect your account → Save.

### 7.4 Gemini API key (HTTP Header Auth — for invoice extraction)

The main pipeline calls Gemini via HTTP request and passes the key as a header.

1. Click **Add Credential → Header Auth**.
2. Set:
   - **Name:** `x-goog-api-key`
   - **Value:** `YOUR_GEMINI_API_KEY` (from Step 2)
3. Save as `Gemini API Key` (or similar).

### 7.5 Google Gemini (PaLM) API credential — for the Chat Agent

1. Click **Add Credential → Google PaLM API**.
2. Paste your **Gemini API Key**.
3. Save as `Google Gemini(PaLM) Api account` (or similar).

---

## Step 8 — Update Workflow Placeholders

Open each workflow in n8n and replace every value marked `REPLACE_HERE` or highlighted below.

### 8.1 Invoice Processing — Google Drive to Sheets

| Node | Field | Replace with |
|------|-------|-------------|
| **Watch Invoice Folder** | Folder ID | Your Google Drive folder ID (Step 5) |
| **Watch Invoice Folder** | Credential | Select your Google Drive OAuth2 credential |
| **Download File** | Credential | Select your Google Drive OAuth2 credential |
| **AI — Structure PDF Invoice** | Credential | Select your Header Auth (Gemini) credential |
| **AI — Extract Image Invoice** | Credential | Select your Header Auth (Gemini) credential |
| **Save to Google Sheets** | Document ID | Your Google Sheet ID (Step 4.3) |
| **Save to Google Sheets** | Credential | Select your Google Sheets OAuth2 credential |
| **Gmail — Invoice Processed** | Send To | Your Gmail address |
| **Gmail — Invoice Processed** | Credential | Select your Gmail OAuth2 credential |
| **Gmail — Extraction Failed** | Send To | Your Gmail address |
| **Gmail — Extraction Failed** | Credential | Select your Gmail OAuth2 credential |

> The Google Sheets link inside the Gmail message body also contains the hardcoded Sheet ID. Update it inside the **Message** field of both Gmail nodes.

### 8.2 Invoice Data Reader

| Node | Field | Replace with |
|------|-------|-------------|
| **Read Invoice Sheet** | Document ID | Your Google Sheet ID (Step 4.3) |
| **Read Invoice Sheet** | Credential | Select your Google Sheets OAuth2 credential |

### 8.3 Invoice Query Agent

| Node | Field | Replace with |
|------|-------|-------------|
| **Google Gemini Chat Model** | Credential | Select your Google PaLM API credential |
| **Read Invoices Tool** | Workflow | Select `Invoice Data Reader` (or paste its workflow ID from Step 6) |

---

## Step 9 — Activate & Test

### 9.1 Activate the workflows

1. Open **Invoice Data Reader** → toggle **Active** (top-right) → Save.
2. Open **Invoice Processing — Google Drive to Sheets** → toggle **Active** → Save.
3. Open **Invoice Query Agent** → toggle **Active** → Save.

### 9.2 Test the Drive trigger

1. Upload a PDF or image invoice to your Google Drive Invoices folder.
2. Within a minute, n8n will detect it, extract the data, save it to Google Sheets, and send you a notification email.
3. Check the **Executions** tab in n8n to see each step and debug any errors.

### 9.3 Test the form upload

n8n exposes a form at:
```
http://localhost:5678/form/upload-invoice
```
Upload any PDF or image invoice to test the form path.

### 9.4 Test the chat agent

1. Open **Invoice Query Agent** in n8n.
2. Click the **Chat** button in the bottom-right corner.
3. Ask a question like:
   - *"What is the total amount of all invoices?"*
   - *"List all invoices from Acme Corp."*
   - *"Which invoices are overdue?"*

---

## Workflow Reference

### Credential IDs in the JSON files

When you import a workflow JSON, n8n assigns new credential IDs specific to your instance. The JSON files in this repo use placeholder values — **do not copy the credential IDs from the JSON directly**. Configure credentials via the n8n UI as described in Step 7.

### Values you MUST update in the JSON files before importing

All user-specific values in the JSON files are marked with `REPLACE_HERE` comments in the `README` table above and with placeholder text directly in the JSON. Search for the following strings after importing:

| Placeholder | Replace with |
|-------------|-------------|
| `YOUR_GOOGLE_DRIVE_FOLDER_ID` | Google Drive folder ID (Step 5) |
| `YOUR_GOOGLE_SHEET_ID` | Google Spreadsheet ID (Step 4.3) |
| `YOUR_EMAIL@gmail.com` | Your Gmail address |
| `YOUR_INVOICE_DATA_READER_WORKFLOW_ID` | Workflow ID from Step 6 |

---

## Troubleshooting

**Workflow does not trigger on new files**
- Confirm the workflow is **Active**.
- Confirm the folder ID in the **Watch Invoice Folder** node matches your Drive folder.
- The trigger polls every minute — wait up to 60 seconds.

**Gemini returns no content / API errors**
- Double-check the API key in the Header Auth credential.
- Ensure the Generative Language API is enabled in your Google Cloud project.
- Check your Gemini free-tier quota at [Google AI Studio](https://aistudio.google.com/).

**Google OAuth errors**
- Make sure `http://localhost:5678/rest/oauth2-credential/callback` is listed as an authorized redirect URI in your Google Cloud OAuth client.
- Ensure all three APIs (Drive, Sheets, Gmail) are enabled in the Cloud project.

**Binary data / image extraction fails**
- Ensure `N8N_DEFAULT_BINARY_DATA_MODE=default` is set in your `.env` file and the container was restarted after the change.

**Invoice Data Reader not found by Agent**
- Import `Invoice Data Reader.json` first, then import `Invoice Query Agent.json`.
- Open the **Read Invoices Tool** node and re-select the `Invoice Data Reader` workflow from the dropdown.

**Emails not sending**
- Verify the Gmail OAuth credential is connected and the **Send To** field contains your email.
- Gmail OAuth requires the Gmail API to be enabled in your Google Cloud project.
