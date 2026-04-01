# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

LinkedIn Learning course: **Hands-On AI: Building AI Agents with Model Context Protocol (MCP) and Agent2Agent (A2A)**. The repo is organized by chapter, each building on the previous to create a multi-agent HR assistant system.

## Setup

```bash
pip install -r requirements.txt
```

Requires a `.env` file with Azure OpenAI credentials:
```
ENDPOINT_URL=...
DEPLOYMENT_NAME=gpt-4.1
API_VERSION=2025-01-01-preview
AZURE_OPENAI_API_KEY=...
PROTOCOL_BUFFERS_PYTHON_IMPLEMENTATION=python
```

## Running the Code

**Chapter 2** — MCP Resources (PDF exposure via stdio):
```bash
python chapter2/code_of_conduct_client.py
```

**Chapter 3** — MCP Tools + RAG (HR policy queries):
```bash
python chapter3/hr_policy_agent.py
```

**Chapter 4** — Stateful MCP over HTTP (timeoff management):
```bash
# Terminal 1
python chapter4/timeoff_db_server.py
# Terminal 2
python chapter4/timeoff_agent.py
```

**Chapter 6** — Multi-agent A2A routing (requires 3 terminals):
```bash
python chapter6/a2a_wrapper_hr_policy_agent.py   # port 9001
python chapter6/a2a_wrapper_timeoff_agent.py      # port 9002
python chapter6/a2a_client_router_agent.py        # router + chat loop
```

## Architecture

The codebase progresses through three architectural patterns:

### MCP Client-Server (Chapters 2–4)
Each chapter has a `*_server.py` (MCP server) and a `*_client.py` or `*_agent.py` (MCP client wrapping a LangGraph ReAct agent). Chapter 2–3 use stdio transport; Chapter 4 uses streamable HTTP.

### RAG Pattern (Chapter 3)
`hr_policy_server.py` loads `hr_policy_document.pdf`, splits it, embeds with HuggingFace `sentence-transformers/all-MiniLM-L6-v2`, and stores in `InMemoryVectorStore`. The `query_policies` MCP tool performs similarity search (k=3). The agent also loads a dynamic MCP prompt via `load_mcp_prompt()`.

### A2A Multi-Agent Routing (Chapter 6)
- Two A2A wrapper agents expose the chapter 3/4 agents as HTTP services (ports 9001, 9002)
- A LangGraph router agent classifies user input → "POLICY", "TIMEOFF", or "UNSUPPORTED"
- Conditional edges route to the appropriate A2A client node, which calls the remote agent via `a2a-sdk`
- Router state uses `add_messages` reducer: `{"messages": [AnyMessage]}`

### Key Technology Roles
| Library | Role |
|---|---|
| `fastmcp` | MCP server definition (resources, tools, prompts) |
| `langchain-mcp-adapters` | `load_mcp_tools()` / `load_mcp_prompt()` bridges MCP → LangChain |
| `langgraph` | `create_react_agent` for chapter agents; `StateGraph` for router |
| `a2a-sdk` | `A2AStarletteApplication` wraps agents; `A2AClient` calls them |
| `langchain-openai` | `AzureChatOpenAI` — the only supported LLM provider |

## Important Notes

- All agent entry points use `async def` and `asyncio.run()`.
- Chapter 6 requires all three processes running simultaneously; agents on 9001/9002 must be up before starting the router.
- The SQLite database in Chapter 4 is in-memory and seeded fresh on each server start.
- This is a read-only course repo — PRs are not accepted.
