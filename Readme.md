# Multi-Agent Civic Helpdesk System with Google ADK

This document details the setup, challenges, and solutions encountered while building a multi-agent civic helpdesk system using the Google Agent Development Kit (ADK) within a Google Cloud Shell environment.

## 1. Introduction

The goal was to create a multi-agent system consisting of an **Orchestrator Agent** and several **Specialist Agents** (Waste, Parking, Housing) to handle civic inquiries. The Orchestrator is responsible for classifying user requests, routing them to the appropriate specialist, merging responses, and handling general inquiries or emergencies directly. All agents were to be defined using YAML files, adhering to Cloud-Only Execution Rules.

## 2. Project Setup

The project was initialized in Google Cloud Shell. A directory structure was established as follows:

service-concierge-agent/
├── agent/
│ ├── init .py
│ └── agent.py # (Initially used for single agent, later replaced by multi_agents/root_agent.yaml)
└── multi_agents/
├── root_agent.yaml # Orchestrator Agent (originally orchestrator.yaml)
├── waste_specialist.yaml
├── parking_specialist.yaml
└── housing_specialist.yaml


### Initial Agent Definition (`agent/agent.py`)

Initially, a single `service_concierge` agent was defined in `agent/agent.py` to demonstrate basic agent functionality and tool integration (`get_current_time`).

### Multi-Agent System Definition (`multi_agents/` directory)

YAML files were created for each agent:
*   **`orchestrator.yaml` (later renamed to `root_agent.yaml`):** Defines the orchestrator's role, classification categories, routing logic, and expected output structure. It includes an `invoke_agent` tool to call specialist agents.
*   **`waste_specialist.yaml`:** Handles waste management inquiries, providing structured responses or escalating if out of scope.
*   **`parking_specialist.yaml`:** Handles parking-related inquiries, providing structured responses or escalating if out of scope.
*   **`housing_specialist.yaml`:** Handles housing-related inquiries, providing structured responses or escalating if out of scope.

## 3. Challenges & Solutions

Several challenges were encountered during the setup and configuration of the ADK multi-agent system.

### Challenge 1: ADK Web UI Not Showing Agent (Initial `service_concierge` Agent)

**Problem:** After defining the `service_concierge` agent in `agent/agent.py`, the ADK web UI did not display it. The `uv run adk web` command was used, but the agent was not visible.

**Root Cause:** The `__init__.py` file in the `agent` directory was not correctly exposing the `root_agent` instance for the ADK framework to discover. The ADK web UI expects a list of agent instances named `agents` in `__init__.py`.

**Solution:** Modified `agent/__init__.py` to explicitly import `root_agent` from `agent.py` and assign it to an `agents` list:

```python
# agent/__init__.py
from .agent import root_agent

agents = [root_agent]


Challenge 2: Persistent "Not Showing" / Cloud Console Confusion
Problem: Despite correcting __init__.py , the user repeatedly reported the agent was "still not showing" and was viewing the "IAM & Admin" page in the Google Cloud Console.

Root Cause: The user was mistakenly looking at the Google Cloud Console instead of the separate browser tab/window where the ADK web interface loads. There was a misunderstanding of how the Cloud Shell "Web Preview" functions.

Solution: Repeated and detailed instructions were provided on how to:

Ensure uv run adk web was running.
Click the "Web Preview" button in Cloud Shell.
Select "Preview on port 8000" to open the ADK web interface in a new browser tab .
Challenge 3: ValueError: No root_agent found for 'multi_agents'
Problem: When attempting to load the multi-agent system using uv run adk web multi_agents --port 8000 , an error occurred: ValueError: No root_agent found for 'multi_agents'. Searched in 'multi_agents.agent.root_agent', 'multi_agents.root_agent' and 'multi_agents/root_agent.yaml'.

Root Cause: The adk web <DIRECTORY> command expects the specified directory to contain either a file named root_agent.yaml or a Python file (e.g., agent.py ) that defines a root_agent variable. The orchestrator agent's YAML file was named orchestrator.yaml , which did not match ADK's expected entry point naming convention for a directory-based load.

Solution: Renamed the orchestrator agent's YAML file from multi_agents/orchestrator.yaml to multi_agents/root_agent.yaml . This allowed the adk web multi_agents command to correctly identify and load the orchestrator as the primary agent.

Challenge 4: Orchestrator Not Returning Specialist Response
Problem: After successfully loading the orchestrator, when a query like "My recycling wasn’t collected. What do I do?" was given, the orchestrator's response in the ADK UI was merely: ``waste_specialist with query: "My recycling wasn’t collected. What do I do?" , instead of the actual structured response from the `waste_specialist`.

Root Cause: The orchestrator's instruction was designed to identify the specialist and indicate the routing, but it lacked the explicit mechanism (a tool) to invoke that specialist and then return its output. The orchestrator was simply stating its intention rather than executing it.

Solution:

Added an invoke_agent tool to the root_agent.yaml (orchestrator's definition). This tool allows the orchestrator to programmatically call other agents.
tools:
  - name: invoke_agent
    description: Call another agent by its name and pass a query to it.
    parameters:
      type: object
      properties:
        agent_name:
          type: string
          description: The name of the agent to invoke (e.g., "waste_specialist", "parking_specialist", "housing_specialist").
        query:
          type: string
          description: The query or message to pass to the invoked agent.
      required:
        - agent_name
        - query


Updated the orchestrator's instruction to explicitly use the invoke_agent tool when routing to a specialist and to return the specialist's response.
4. Current State
The multi-agent system is now configured with:

An orchestrator agent (defined in multi_agents/root_agent.yaml ) capable of classifying requests, routing to specialists using the invoke_agent tool, handling general inquiries, and responding to emergencies.
Three specialist agents ( waste_specialist , parking_specialist , housing_specialist ) (defined in their respective YAML files in multi_agents/ ) that handle domain-specific queries and escalate out-of-scope requests.
The system is runnable via uv run adk web multi_agents --port 8000 from the project root.
5. Next Steps
The following tasks remain to be completed:

Test Scenarios: Execute and verify all five required test scenarios in the ADK web UI, capturing screenshots for correct routing, clarifying questions, and final responses.
Mandatory Debugging Exercise: Intentionally introduce and fix one failure (e.g., YAML indentation error, incorrect routing logic), documenting the failure, trace, root cause, exact change, and successful re-run.


<!--<serializedDesign>{}</serializedDesign>-->

