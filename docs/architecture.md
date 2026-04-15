# Architecture

Two design patterns make this agent distinct from a standard ReAct loop.

## 1. Sub-agent context isolation

When the parent agent delegates a research task, it calls `_create_task_tool` which spawns a sub-agent with a **clean context window** containing only the task description — no parent message history.

```python
# In task.py — the key line
state["messages"] = [{"role": "user", "content": description}]
result = sub_agent.invoke(state)
```

Why this matters: a long parent conversation would otherwise pollute the sub-agent's reasoning. By clearing `state["messages"]` before invocation, each sub-agent reasons about exactly one task in isolation. The parent receives only the sub-agent's final message as a `ToolMessage`, hiding all intermediate tool calls.

## 2. Virtual file system + context offloading

The `DeepAgentState` carries a `files` dict (filename → content) with a custom `file_reducer` that merges updates additively:

```python
files: Annotated[NotRequired[dict[str, str]], file_reducer]
```

When `tavily_search` runs, it saves the full raw webpage content to a file in state and returns only a short summary (≤150 words) to the message thread. The parent agent never sees the full content unless it explicitly calls `read_file()`.

This has two benefits:
- **Token efficiency**: the parent's context stays small even after many searches
- **Selective retrieval**: the parent can read specific files when it needs detail, mirroring how a human researcher skims abstracts before reading a paper

The `file_reducer` ensures that file writes from parallel sub-agents are merged correctly rather than overwriting each other.
