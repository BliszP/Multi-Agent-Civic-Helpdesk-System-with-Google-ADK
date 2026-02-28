# Complete Setup Guide — Multi-Agent Civic Helpdesk (Google ADK)

This guide covers every step from zero to a running multi-agent system:
Google Cloud project → AI Studio API key → Cloud Shell environment → GitHub → running agents.

---

## PART 1 — Google Cloud Project Setup

### Step 1: Create a Google Cloud Project

1. Open [https://console.cloud.google.com](https://console.cloud.google.com)
2. Click the **project selector dropdown** at the very top of the page
3. Click **New Project** (top-right of the dialog)
4. Fill in:
   - **Project name:** `civic-helpdesk` (or any name you prefer)
   - **Organisation:** leave as default unless you have one
5. Click **Create**
6. Wait ~30 seconds, then select your new project from the dropdown
7. Note your **Project ID** shown under the project name (e.g. `civic-helpdesk-419301`) — you will need this

### Step 2: Open Cloud Shell

1. In the Google Cloud Console, click the **Cloud Shell icon** (terminal icon `>_`) in the top-right toolbar
2. A terminal pane opens at the bottom of the browser — this is your Cloud Shell
3. Authenticate and set your project:

```bash
gcloud auth login
# Follow the browser prompt to authenticate

gcloud config set project YOUR_PROJECT_ID
# Replace YOUR_PROJECT_ID with the ID from Step 1
```

### Step 3: Enable Required APIs

```bash
gcloud services enable aiplatform.googleapis.com
```

Wait for the command to complete (10–30 seconds).

---

## PART 2 — Get a Gemini API Key from Google AI Studio

This links your Google Cloud project to the Gemini model your agents use.

### Step 4: Import Your Project into AI Studio

1. Go to [https://aistudio.google.com](https://aistudio.google.com)
2. Sign in with the **exact same Google account** you used for your Cloud project
3. In the left sidebar or top menu, click **Get API Key**
4. On the API Keys page, click **+ Create API key**
5. In the dialog, select **Create API key in existing project**
6. From the dropdown, select your project (e.g. `civic-helpdesk-419301`)
7. Click **Create API key**
8. **Copy the key immediately** and store it somewhere safe (password manager, secure note)

> ⚠️ You will not be able to view the full key again after closing this dialog.
> ⚠️ Never paste your API key into any file you commit to GitHub.

---

## PART 3 — GitHub Repository Setup

### Step 5: Create a GitHub Repository

1. Go to [https://github.com](https://github.com) and sign in
2. Click **+** → **New repository**
3. Fill in:
   - **Repository name:** `Multi-Agent-Civic-Helpdesk-System-with-Google-ADK`
   - **Visibility:** Public or Private
   - **DO NOT** tick "Add a README file" if you already have one locally — this avoids merge conflicts on first push
4. Click **Create repository**
5. Copy the SSH URL shown (e.g. `git@github.com:YourUsername/Multi-Agent-Civic-Helpdesk-System-with-Google-ADK.git`)

### Step 6: Configure Git in Cloud Shell

```bash
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"
```

### Step 7: Set Up SSH Key for GitHub (Cloud Shell)

```bash
# Generate SSH key
ssh-keygen -t ed25519 -C "your.email@example.com"
# Press Enter three times to accept defaults and skip passphrase

# Display your public key
cat ~/.ssh/id_ed25519.pub
```

Copy the entire output. Then:

1. Go to GitHub → **Settings** → **SSH and GPG keys**
2. Click **New SSH key**
3. Title: `Google Cloud Shell`
4. Paste your public key
5. Click **Add SSH key**

Test the connection:

```bash
ssh -T git@github.com
# You should see: "Hi YourUsername! You've successfully authenticated..."
```

---

## PART 4 — Project Installation in Cloud Shell

### Step 8: Clone or Initialise the Project

**If starting from scratch (no existing repo):**

```bash
mkdir service-concierge-agent
cd service-concierge-agent
git init
git remote add origin git@github.com:YourUsername/Multi-Agent-Civic-Helpdesk-System-with-Google-ADK.git
```

**If cloning an existing repo:**

```bash
git clone git@github.com:YourUsername/Multi-Agent-Civic-Helpdesk-System-with-Google-ADK.git
cd Multi-Agent-Civic-Helpdesk-System-with-Google-ADK
```

### Step 9: Install uv and Google ADK

```bash
# Install uv package manager
pip install uv

# Initialise uv project (skip if pyproject.toml already exists)
uv init

# Install Google ADK
uv add google-adk

# Verify installation
uv run python -c "import google.adk; print('ADK installed successfully')"
```

### Step 10: Create the Project Directory Structure

```bash
mkdir -p multi_agents
```

Create the agent files in order:

**`multi_agents/root_agent.yaml`** (Orchestrator — must be named exactly this):

```yaml
# yaml-language-server: $schema=https://raw.githubusercontent.com/google/adk-python/refs/heads/main/src/google/adk/agents/config_schemas/AgentConfig.json
name: orchestrator
description: Routes user requests to appropriate specialist agents for a civic helpdesk.
model: gemini-2.5-flash
instruction: |
  You are the central orchestrator for a civic helpdesk. Classify each request
  and transfer to the correct specialist, or handle directly if no specialist applies.

  Classification:
  - Waste Management → transfer to waste_specialist
  - Parking → transfer to parking_specialist
  - Housing → transfer to housing_specialist
  - General Inquiry → handle yourself, ask clarifying questions
  - Emergency/Safety → respond with emergency escalation, do NOT transfer

  For direct responses use this structure:
  Summary:
  What I need from you:
  Steps to resolve:
  Edge cases / warnings:
  Next possible help:

sub_agents:
  - config_path: waste_specialist.yaml
  - config_path: parking_specialist.yaml
  - config_path: housing_specialist.yaml
```

**`multi_agents/waste_specialist.yaml`:**

```yaml
name: waste_specialist
description: Handles waste management, recycling, and rubbish collection inquiries.
model: gemini-2.5-flash
instruction: |
  You are a waste management specialist for a civic helpdesk.
  Handle queries about recycling, missed collections, bin types, and collection schedules.
  If a query is outside waste management, say it is out of scope and suggest the user
  contact the helpdesk for re-routing.

  Always respond with this structure:
  Summary:
  What I need from you:
  Steps to resolve:
  Edge cases / warnings:
  Next possible help:
```

**`multi_agents/parking_specialist.yaml`:**

```yaml
name: parking_specialist
description: Handles parking permits, fines, zones, and parking-related inquiries.
model: gemini-2.5-flash
instruction: |
  You are a parking specialist for a civic helpdesk.
  Handle queries about resident permits, visitor permits, parking fines, PCNs,
  parking zones, and disabled bays.
  If a query is outside parking, say it is out of scope and suggest re-routing.

  Always respond with this structure:
  Summary:
  What I need from you:
  Steps to resolve:
  Edge cases / warnings:
  Next possible help:
```

**`multi_agents/housing_specialist.yaml`:**

```yaml
name: housing_specialist
description: Handles council housing, repairs, damp, tenancy, and housing inquiries.
model: gemini-2.5-flash
instruction: |
  You are a housing specialist for a civic helpdesk.
  Handle queries about council housing repairs, damp and mould, tenancy agreements,
  rent, anti-social behaviour, and housing applications.
  If a query is outside housing, say it is out of scope and suggest re-routing.

  Always respond with this structure:
  Summary:
  What I need from you:
  Steps to resolve:
  Edge cases / warnings:
  Next possible help:
```

---

## PART 5 — Environment & API Key Configuration

### Step 11: Set the API Key as an Environment Variable

```bash
export GOOGLE_API_KEY="paste_your_key_here"
```

To persist it across Cloud Shell sessions:

```bash
echo 'export GOOGLE_API_KEY="paste_your_key_here"' >> ~/.bashrc
source ~/.bashrc
```

Verify it is set:

```bash
echo $GOOGLE_API_KEY
# Should print your key (not blank)
```

### Step 12: Create a `.gitignore` to Protect Your Key

```bash
cat > .gitignore << 'EOF'
.env
*.env
__pycache__/
.venv/
*.pyc
.adk/
EOF
```

> Never store your API key in a `.env` file that gets committed. Use environment variables only.

---

## PART 6 — Running the System

### Step 13: Run the Multi-Agent System

```bash
uv run adk web multi_agents --port 8000
```

### Step 14: Open the ADK Web UI

1. In Cloud Shell, click the **Web Preview** button (square with arrow icon, top-right of the terminal toolbar)
2. Select **Preview on port 8000**
3. A new browser tab opens with the ADK chat interface
4. Select `orchestrator` from the agent dropdown if prompted
5. Start testing with the scenarios below

> ℹ️ If you see `ValueError: No root_agent found`, check that your orchestrator file is named exactly `root_agent.yaml`, not `orchestrator.yaml` or anything else.

---

## PART 7 — GitHub Push Workflow

### Step 15: Initial Push

```bash
cd ~/service-concierge-agent   # or your project directory

git add .
git commit -m "Initial commit: multi-agent civic helpdesk with Google ADK"
git push -u origin main
```

### Step 16: If Push Is Rejected (Non-Fast-Forward)

This happens when GitHub initialised the repo with a README that your local history doesn't include:

```bash
git pull origin main --allow-unrelated-histories
# If Git opens a merge editor: save and exit (Esc → :wq in vim, Ctrl+X in nano)

git push -u origin main
```

### Step 17: Subsequent Pushes

```bash
git add .
git commit -m "your commit message"
git push
```

---

## PART 8 — Troubleshooting Reference

| Error | Cause | Fix |
|---|---|---|
| `ValueError: No root_agent found` | Orchestrator YAML not named `root_agent.yaml` | Rename file to `root_agent.yaml` |
| `extra_forbidden` on `tools` | ADK YAML rejects `description`/`parameters` in `function` blocks | Remove custom tool; use `sub_agents` with `config_path` |
| `extra_forbidden` on `agent_refs` | `agent_refs` is not a valid ADK YAML field | Use `sub_agents` instead |
| Agent not visible in ADK UI | `__init__.py` not exposing agent correctly | Add `from .agent import root_agent` and `agents = [root_agent]` |
| Git push rejected | Remote has commits not in local history | `git pull origin main --allow-unrelated-histories` |
| `GOOGLE_API_KEY` not found | Environment variable not set | `export GOOGLE_API_KEY="your_key"` |
| Port 8000 already in use | Previous `adk web` still running | `pkill -f "adk web"` then re-run |

---

## Quick Reference — Key Commands

```bash
# Run agents
uv run adk web multi_agents --port 8000

# Kill running server
pkill -f "adk web"

# Check API key is set
echo $GOOGLE_API_KEY

# Git workflow
git add .
git commit -m "message"
git push

# Pull with unrelated histories (first push fix)
git pull origin main --allow-unrelated-histories
```
