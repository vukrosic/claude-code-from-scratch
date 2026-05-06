# Week 01: Build The First Coding Agent Turn Loop

This week teaches the first habit of building a Claude Code-style coding agent:

```text
a coding agent is a stateful loop, not just a chatbot response
```

You will build the smallest useful mental model of a coding agent turn. No file
editing yet. No shell execution yet. The goal is to understand the loop that all
later features plug into.

Work through this file from top to bottom. Every task appears inside the lesson
at the moment you need it.

## Step 1: Understand The Job Of Week 01

A normal chatbot can answer a question from text. A coding agent has to keep
track of more state:

```text
1. what the user asked for
2. what repo context is available
3. which tools might be useful
4. what happened during this turn
5. why the turn stopped
6. what should be remembered next time
```

This week, we build the first version of that shape:

```text
prompt -> route -> turn result -> transcript
```

That is the core loop. Everything else in the course makes this loop more real.

## Step 2: Learn The Five Objects

The first version of the agent needs five objects:

```text
Prompt:
    the user's request

Tool:
    a named capability the agent may use later

Route:
    the agent's guess about which tools matter

TurnResult:
    the result of one agent turn

Transcript:
    the memory trail across turns
```

Keep them small. Good coding-agent systems start boring and explicit.

## Step 3: Write The Smallest Data Model

Start with plain Python dataclasses:

```python
from dataclasses import dataclass, field


@dataclass(frozen=True)
class Tool:
    name: str
    description: str


@dataclass(frozen=True)
class Route:
    prompt: str
    matched_tools: tuple[str, ...]


@dataclass(frozen=True)
class TurnResult:
    prompt: str
    message: str
    matched_tools: tuple[str, ...]
    stop_reason: str


@dataclass
class Transcript:
    entries: list[str] = field(default_factory=list)

    def append(self, text: str) -> None:
        self.entries.append(text)

    def replay(self) -> tuple[str, ...]:
        return tuple(self.entries)
```

Read this as the first version of memory:

```text
each turn can append what happened
later turns can replay the previous entries
```

That tiny transcript will become important once the agent edits files, runs
tests, and needs to explain what it already tried.

## Step 4: Add A Tiny Tool Inventory

A Claude Code-style agent has a tool surface. The first version can be only
metadata:

```python
TOOLS = (
    Tool(
        name="read_file",
        description="Read a file before deciding what to change.",
    ),
    Tool(
        name="edit_file",
        description="Apply a small, reviewable code edit.",
    ),
    Tool(
        name="run_tests",
        description="Run the project test command and report the result.",
    ),
)
```

These tools do not execute yet. Week 01 only teaches selection.

The useful question is:

```text
Given this prompt, which tools sound relevant?
```

That question is routing.

## Step 5: Build A Naive Router

The first router can be simple keyword matching:

```python
def route_prompt(prompt: str, tools: tuple[Tool, ...] = TOOLS) -> Route:
    lowered = prompt.lower()
    matches: list[str] = []

    if any(word in lowered for word in ("read", "inspect", "open", "look")):
        matches.append("read_file")
    if any(word in lowered for word in ("change", "edit", "fix", "add")):
        matches.append("edit_file")
    if any(word in lowered for word in ("test", "pytest", "fail", "verify")):
        matches.append("run_tests")

    return Route(prompt=prompt, matched_tools=tuple(matches))
```

This is not smart. That is fine.

The point is to make routing visible:

```text
prompt:
    "fix the failing test"

matched tools:
    edit_file, run_tests
```

Later, routing can use a model, command registry, permissions, or repo context.
The first version should be easy to understand.

## Step 6: Run One Agent Turn

Now wrap the router in a turn loop:

```python
class CodingAgent:
    def __init__(self) -> None:
        self.transcript = Transcript()

    def run_turn(self, prompt: str) -> TurnResult:
        route = route_prompt(prompt)
        if route.matched_tools:
            tool_text = ", ".join(route.matched_tools)
            message = f"I would start with: {tool_text}"
        else:
            message = "I need more repo context before choosing a tool."

        result = TurnResult(
            prompt=prompt,
            message=message,
            matched_tools=route.matched_tools,
            stop_reason="completed",
        )
        self.transcript.append(f"user: {prompt}")
        self.transcript.append(f"agent: {message}")
        return result
```

This is the first real agent shape:

```text
receive prompt
route prompt
create result
write transcript
return result
```

No magic. Just state moving through a loop.

## Step 7: Try Three Prompts

Use the same agent for multiple turns:

```python
agent = CodingAgent()

print(agent.run_turn("read the README and explain the project"))
print(agent.run_turn("add a repo map command"))
print(agent.run_turn("run tests after the change"))
print(agent.transcript.replay())
```

Expected behavior:

```text
"read the README..." matches read_file
"add a repo map command" matches edit_file
"run tests..." matches run_tests
transcript contains all three user turns and agent messages
```

This is still a toy. But the toy already teaches the core idea: the agent is
not just one answer. It is a stateful workflow.

## Step 8: Understand Stop Reasons

Every turn should say why it stopped.

For now, use:

```text
completed:
    the turn finished normally

needs_context:
    the agent cannot choose a next action yet

blocked:
    the agent knows what to do but is not allowed to do it

error:
    something failed while running the turn
```

Update the no-match branch like this:

```python
if route.matched_tools:
    tool_text = ", ".join(route.matched_tools)
    stop_reason = "completed"
    message = f"I would start with: {tool_text}"
else:
    stop_reason = "needs_context"
    message = "I need more repo context before choosing a tool."
```

Stop reasons make the agent debuggable. Later, they decide whether the loop
continues, asks for permission, runs tests, or reports a failure.

## Step 9: The Week 01 Build

Your build this week is one small file:

```text
claudecode/agent.py
```

It should contain:

```text
Tool
Route
TurnResult
Transcript
TOOLS
route_prompt
CodingAgent
```

Keep the implementation short. The goal is not completeness. The goal is to
make the turn loop concrete.

## Step 10: Done Checklist

You are done when:

- `CodingAgent.run_turn` accepts a prompt
- the prompt is routed to zero or more tool names
- the turn returns a `TurnResult`
- the turn appends user and agent entries to the transcript
- the result includes a `stop_reason`
- you can explain the loop as `prompt -> route -> result -> transcript`

Stop there.

## Skool Submission

Use this format:

```text
Week 01 Submission

My agent loop in one sentence:

The objects I built:

Example prompt:

Matched tools:

What the transcript stores:

One thing I want reviewed:
```
