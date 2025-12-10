# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is an AI-powered expense tracking agent built with the Google Agent Development Kit (ADK). It provides a natural language interface for managing personal finances through conversational commands.

## Development Commands

### Setup and Dependencies
```bash
# Install dependencies using uv (required)
uv sync

# Configure environment (copy and edit .env file)
cp .env.example .env
# Set: GOOGLE_GENAI_USE_VERTEXAI=true, GOOGLE_CLOUD_PROJECT, GOOGLE_CLOUD_LOCATION

# Authenticate with Google Cloud
gcloud auth login
```

### Running Locally
```bash
# Start the agent with web UI (accessible at http://localhost:8080)
uv run adk web

# Alternative: Run the FastAPI server directly
uv run uvicorn server:app --host 0.0.0.0 --port 8080
```

### Deployment
```bash
# Deploy to Google Cloud Run (method 1: gcloud)
gcloud run deploy expense-tracker-agent \
  --source . \
  --region <REGION> \
  --allow-unauthenticated \
  --memory 4Gi \
  --cpu 2

# Deploy to Google Cloud Run (method 2: ADK CLI)
uv run adk deploy cloud_run \
  --project=<PROJECT_ID> \
  --region=<REGION> \
  --service_name=expense-tracker-agent \
  --app_name=expense-tracker \
  --with_ui \
  ./expense_tracker
```

## Architecture

### Core Components

**expense_tracker/agent.py**
- Defines the root agent using Google ADK's `Agent` class
- Uses the `gemini-2.5-flash` model
- Registers all available tools (functions the agent can call)
- Contains the agent's system instruction that defines its behavior and capabilities

**expense_tracker/models.py**
- Pydantic data models for type validation
- `Expense`: Single expense with amount, category, description
- `RecurringExpense`: Repeating expenses with frequency, dates, and unique IDs

**expense_tracker/tools.py**
- Contains all tool functions the agent can invoke
- Uses in-memory databases (`expenses_db`, `recurring_expenses_db`) for storage
- Tools include: add_expense, calculate_total_spending, get_spending_by_category, list_recent_expenses, add_recurring_expense, list_recurring_expenses, project_spending

**server.py**
- FastAPI application that wraps the ADK agent
- Uses `get_fast_api_app()` from ADK to integrate the agent with FastAPI
- In-memory session service (no persistent storage)
- Provides REST API endpoints for interacting with the agent

### Key Design Patterns

**In-Memory Storage**: Both `expenses_db` and `recurring_expenses_db` are Python lists that persist only during runtime. Restarting the server clears all data.

**Tool-Based Architecture**: The agent operates by calling Python functions decorated/registered as tools. Each tool must have clear docstrings as the LLM uses these to understand when to invoke them.

**Date Handling**: Recurring expenses use datetime objects internally but accept string inputs in YYYY-MM-DD format. The `add_recurring_expense` tool calculates the `next_due_date` based on frequency.

**ADK Integration**: The Google ADK handles the agent orchestration, LLM calls, tool execution, and web UI generation. The application just needs to define the agent, tools, and wrap it in FastAPI.

## Project Structure
```
expense_tracker/     # Agent module
├── agent.py        # Agent definition with tools and instructions
├── models.py       # Pydantic data models
└── tools.py        # Tool functions (agent capabilities)
server.py           # FastAPI application entry point
Dockerfile          # Container configuration for deployment
pyproject.toml      # Python project metadata (requires Python 3.12+)
```

## Environment Requirements

- Python 3.12+ (specified in pyproject.toml)
- uv package manager (used for all dependency management)
- Google Cloud SDK (for authentication and deployment)
- Google Cloud project with Vertex AI enabled
