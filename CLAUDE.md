# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

A Manus-style multi-domain research agent built with LangGraph. The parent agent plans research tasks using a TODO list and delegates each task to isolated sub-agents. Sub-agents search the web and offload full content to a virtual file system, returning only AI-generated summaries to the parent — keeping the parent's context small while preserving full content for selective retrieval.

## Setup & Commands

```bash
uv sync                   # Install dependencies
cp .env.example .env      # Configure OPENAI_API_KEY and TAVILY_API_KEY
```

**Linting & formatting:**

```bash
uv run ruff check --fix src/   # Lint with autofix
uv run ruff format src/        # Format
uv run pyright src/            # Type check
```

**Run pre-commit hooks manually:**

```bash
uv run pre-commit run --all-files
```

**Run the demo:**

```bash
uv run jupyter notebook examples/research_demo.ipynb
```

**Direct usage:**

```python
from deep_research_agent.agent import create_deep_research_agent

agent = create_deep_research_agent()
result = agent.invoke({
    "messages": [{"role": "user", "content": "Your research question here"}]
})
```

## Environment Variables

Required in `.env`:

- `OPENAI_API_KEY` — LLM provider
- `TAVILY_API_KEY` — web search

Optional:

- `LANGSMITH_TRACING`, `LANGSMITH_API_KEY`, `LANGSMITH_PROJECT` — tracing

## Architecture

### Agentic pattern

This repo implements the **Coordinator pattern**: the parent agent uses AI reasoning to dynamically decompose a research question and dispatch sub-tasks to specialized sub-agents. The parent maintains overall context; each sub-agent gets a clean, isolated context window (task description only) and is stateless from the parent's perspective — returning only its final message as a `ToolMessage`. The `@tool` wrapping in `task.py` is a LangGraph implementation detail, not an "Agent as a Tool" pattern — the behavioral structure is Coordinator.

### Two core implementation patterns

**1. Sub-agent context isolation** (`src/deep_research_agent/task.py`): When the parent delegates a research task, `_create_task_tool` spawns a sub-agent with a clean context window containing only the task description — no parent message history. The parent receives only the sub-agent's final message as a `ToolMessage`, hiding all intermediate tool calls.

**2. Virtual file system + context offloading** (`src/deep_research_agent/state.py`): `DeepAgentState` carries a `files: dict[str, str]` with a `file_reducer` that merges updates additively. When `tavily_search` runs, it saves full raw webpage content to a file in state and returns only a ≤150-word summary to the message thread. The parent reads files only on-demand via `read_file()`.

### Data flow

```
User → Parent Agent
         ├── write_todos()         # Plans research tasks
         ├── task(description, "research-agent")  # Delegates to sub-agent
         │       └── Sub-agent (isolated context: task description only)
         │               ├── tavily_search() → saves raw content to state["files"]
         │               │                    returns summary to messages
         │               └── think_tool()   → reflection between searches
         │       └── Returns: final sub-agent message as ToolMessage + merged files
         ├── ls() / read_file()    # Selective retrieval from virtual FS
         └── Final response
```

### Module responsibilities

| Module                 | Purpose                                                                                                                            |
| ---------------------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| `agent.py`             | Public factory `create_deep_research_agent()` — assembles tools, prompts, sub-agent configs into a compiled LangGraph graph        |
| `state.py`             | `DeepAgentState` (extends `AgentState` with `todos` and `files`), `file_reducer`, `Todo` TypedDict                                 |
| `task.py`              | `_create_task_tool()` — creates the `task` delegation tool that implements context isolation                                       |
| `tools/research.py`    | `tavily_search` (search + offload), `think_tool` (reflection), `summarize_webpage_content` (uses `gpt-5.4-nano` structured output) |
| `tools/files.py`       | `ls`, `read_file`, `write_file` — virtual filesystem tools over `state["files"]`                                                   |
| `tools/todos.py`       | `write_todos`, `read_todos` — TODO tracking tools over `state["todos"]`                                                            |
| `prompts/system.py`    | All system prompt constants and tool descriptions for the parent agent                                                             |
| `prompts/summarize.py` | Prompts for the research sub-agent and web content summarization                                                                   |

### Key design constraints

- **Tool returns `Command`**: Tools that modify state (`tavily_search`, `write_file`, `write_todos`, `task`) return `langgraph.types.Command` objects — not plain strings — to update state atomically alongside injecting `ToolMessage` into messages.
- **`file_reducer` is additive**: Parallel sub-agents each write distinct files; the reducer merges dicts so no writes are lost.
- **Summarization model**: `tools/research.py` hard-codes `gpt-5.4-nano` for the summarization step (separate from the main agent model).
- **`langgraph.json`**: Points to `create_deep_research_agent` as the graph entrypoint for LangGraph Server deployment (future work).

## Code Style

- Ruff with Google-style docstrings (`pydocstyle` convention)
- Pyright in strict mode for `src/`, basic mode for `tests/`
- `ANN` rules enforced (type annotations required) except in tests
- `T20` rules enforced (no `print` statements — use `rich` or return values)
- Line length: 100 characters
- Python 3.13+
