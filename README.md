# Multi-Agent Civic Helpdesk System with Google ADK

A multi-agent civic helpdesk built with the Google Agent Development Kit (ADK), featuring an Orchestrator Agent that classifies and routes requests to domain-specific Specialist Agents (Waste, Parking, Housing). All agents are defined using YAML configuration files and run entirely in Google Cloud Shell.

---

## Project Structure

```
service-concierge-agent/
├── agent/
│   ├── __init__.py
│   └── agent.py
└── multi_agents/
    ├── root_agent.yaml          # Orchestrator (entry point)
    ├── waste_specialist.yaml
    ├── parking_specialist.yaml
    └── housing_specialist.yaml
```

---

## Prerequisites

- Google Cloud account with a project created
- Google Cloud Shell (browser-based terminal)
- Gemini API key from Google AI Studio

---

## 1. Create & Configure Your Google Cloud Project

### 1.1 Create a Project in Google Cloud Console

1. Go to [https://console.cloud.google.com](https://console.cloud.google.com)
2. Click the project dropdown at the top → **New Project**
3. Enter a project name (e.g. `civic-helpdesk`) → **Create**
4. Note your **Project ID** (e.g. `civic-helpdesk-123456`)

### 1.2 Enable Required APIs

In Cloud Shell or the Console terminal, run:

```bash
gcloud auth login
gcloud config set project YOUR_PROJECT_ID
gcloud services enable aiplatform.googleapis.com
```

---

## 2. Get Your Gemini API Key from Google AI Studio

This is how you connect your Cloud project to the Gemini model your agents will use.

1. Go to [https://aistudio.google.com](https://aistudio.google.com)
2. Click **Sign in** — use the **same Google account** as your Cloud project
3. In the top-right corner, click **Get API Key**
4. Click **Create API key in existing project**
5. Select your project from the dropdown (e.g. `civic-helpdesk-123456`)
6. Click **Create API key** — copy and save the key securely

> ⚠️ Never commit your API key to GitHub. Store it as an environment variable (see Step 5).

---

## 3. Clone the Repository in Google Cloud Shell

Open [Google Cloud Shell](https://shell.cloud.google.com) and run:

```bash
git clone https://github.com/BliszP/Multi-Agent-Civic-Helpdesk-System-with-Google-ADK.git
cd Multi-Agent-Civic-Helpdesk-System-with-Google-ADK
```

---

## 4. Install Dependencies

This project uses `uv` as the Python package manager:

```bash
# Install uv if not already available
pip install uv

# Initialise the project and install google-adk
uv init
uv add google-adk
```

---

## 5. Set Your API Key

```bash
export GOOGLE_API_KEY="your_gemini_api_key_here"
```

To make this persist across Cloud Shell sessions, add it to your shell profile:

```bash
echo 'export GOOGLE_API_KEY="your_gemini_api_key_here"' >> ~/.bashrc
source ~/.bashrc
```

---

## 6. Run the Multi-Agent System

```bash
uv run adk web multi_agents --port 8000
```

Then in Cloud Shell, click **Web Preview** → **Preview on port 8000** to open the ADK UI in your browser.

> ℹ️ The ADK framework discovers the orchestrator automatically because the entry point is named `root_agent.yaml`. Any other name will cause a `ValueError: No root_agent found` error.

---

## 7. Agent Architecture

### Orchestrator (`root_agent.yaml`)

The central router. Classifies every incoming request into one of five categories and acts accordingly:

| Category | Action |
|---|---|
| Waste Management | Transfers to `waste_specialist` |
| Parking | Transfers to `parking_specialist` |
| Housing | Transfers to `housing_specialist` |
| General Inquiry | Handles directly, asks for clarification |
| Emergency / Safety | Responds with emergency escalation, no transfer |

Sub-agents are registered using the `sub_agents` field with `config_path` references — the correct ADK YAML pattern for multi-agent routing:

```yaml
sub_agents:
  - config_path: waste_specialist.yaml
  - config_path: parking_specialist.yaml
  - config_path: housing_specialist.yaml
```

> ⚠️ Do not use a custom `invoke_agent` function tool — ADK does not permit `description` or `parameters` as extra fields inside a `function` tool entry in YAML (`extra_forbidden` Pydantic error). Use `sub_agents` instead; ADK auto-generates transfer tools from them.

### Specialist Agents

Each specialist handles one domain and returns a structured response:

```
Summary:
What I need from you:
Steps to resolve:
Edge cases / warnings:
Next possible help:
```

If a query is out of scope, specialists escalate back to the orchestrator.

---

## 8. Test Scenarios

Run each scenario in the ADK web UI and capture the Trace/Events panel as evidence.

| # | Input | Expected Routing | Expected Behaviour |
|---|---|---|---|
| 1 | `My recycling wasn't collected. What do I do?` | `waste_specialist` | Structured missed collection response |
| 2 | `How do I apply for a resident parking permit?` | `parking_specialist` | Structured permit application steps |
| 3 | `There is damp and mould in my council flat.` | `housing_specialist` | Structured damp/mould reporting steps |
| 4 | `I got a letter from the council and I don't understand it.` | None (orchestrator handles) | Clarifying questions, no specialist call |
| 5 | `Someone is in danger.` | None (orchestrator handles) | Emergency escalation language, no specialist call |

---

## 9. Mandatory Debugging Exercise

Intentionally introduce one failure, document it, fix it, and re-run.

**Example failure to introduce:** Add an incorrect indentation in `root_agent.yaml` under `sub_agents`.

Document:
- The exact change made
- The error trace produced
- The root cause identified
- The exact fix applied
- Screenshot of successful re-run

---

## 10. Known Challenges & Solutions

| Challenge | Root Cause | Solution |
|---|---|---|
| Agent not visible in ADK UI | `__init__.py` not exposing `root_agent` correctly | Added `from .agent import root_agent` and `agents = [root_agent]` |
| `ValueError: No root_agent found` | Orchestrator YAML was named `orchestrator.yaml` | Renamed to `root_agent.yaml` |
| `extra_forbidden` Pydantic error on `tools` | ADK YAML does not allow `description`/`parameters` inside `function` tool blocks | Removed custom tool; used `sub_agents` with `config_path` instead |
| `agent_refs` field rejected | `agent_refs` is not a valid `AgentConfig` field | Replaced with `sub_agents` |
| Git push rejected (non-fast-forward) | Remote had commits (auto-created README) not in local history | Used `git pull origin main --allow-unrelated-histories` then pushed |

---

## License

Apache 2.0
