# Week 01: Build The First Coding Agent Turn Loop

This week teaches the first habit of building a Claude Code-style coding agent:

```text
the model chooses tools, the runtime controls what actually happens
```

You will build the smallest useful mental model of a coding agent turn. No real
LLM API yet. No file editing yet. The goal is to understand the loop that all
later features plug into.

Work through this file from top to bottom. Every task appears inside the lesson
at the moment you need it.

## Step 1: Understand The Job Of Week 01

A Claude Code-style agent is not a keyword router.

In the real architecture, the runtime gives the model:

```text
1. the user's prompt
2. the available tool names
3. the available tool descriptions
4. the tool argument schemas
5. relevant repo context
6. prior transcript entries
```

Then the model decides whether to answer directly or request a tool call.

The runtime still stays in charge:

```text
prompt -> model decision -> validate tool call -> result -> transcript
```

This week, we fake the model decision so we can build the runtime loop first.
Later, the fake model gets replaced with a real LLM tool-calling API.

## Step 2: Learn The Five Objects

The first version of the agent needs five objects:

```text
ToolSpec:
    the name and description shown to the model

ToolCall:
    the specific tool and arguments requested by the model

ModelDecision:
    the model's answer, with or without a tool call

TurnResult:
    the runtime's final result for one turn

Transcript:
    the memory trail across turns
```

This is the right separation:

```text
the model proposes
the runtime validates, executes, and records
```

## Step 3: Write The Smallest Data Model

Start with plain Python dataclasses:

```python
from dataclasses import dataclass, field


@dataclass(frozen=True)
class ToolSpec:
    name: str
    description: str


@dataclass(frozen=True)
class ToolCall:
    name: str
    arguments: dict[str, str]


@dataclass(frozen=True)
class ModelDecision:
    message: str
    tool_call: ToolCall | None = None


@dataclass(frozen=True)
class TurnResult:
    prompt: str
    message: str
    tool_call: ToolCall | None
    stop_reason: str


@dataclass
class Transcript:
    entries: list[str] = field(default_factory=list)

    def append(self, role: str, text: str) -> None:
        self.entries.append(f"{role}: {text}")

    def replay(self) -> tuple[str, ...]:
        return tuple(self.entries)
```

Read this as the first version of memory:

```text
each turn can append user and agent messages
later turns can replay what already happened
```

That tiny transcript will matter once the agent edits files, runs tests, and
needs to explain what it already tried.

## Step 4: Add Tool Specs

Tool specs are what the model sees.

Start with metadata only:

```python
TOOLS = (
    ToolSpec(
        name="read_file",
        description="Read a file from the repo before deciding what to change.",
    ),
    ToolSpec(
        name="edit_file",
        description="Apply a small, reviewable edit to a repo file.",
    ),
    ToolSpec(
        name="run_tests",
        description="Run the project test command and summarize the result.",
    ),
)
```

These tools do not execute yet. Week 01 teaches the tool-calling shape.

The useful question is not:

```text
Which keyword did the prompt contain?
```

The useful question is:

```text
Given the prompt and tool specs, what action does the model request?
```

## Step 5: Mock The Model Decision

In production, an LLM chooses from the tool specs.

In Week 01, use a mock function with hardcoded examples:

```python
def mock_model_decide_next_action(
    prompt: str,
    tools: tuple[ToolSpec, ...] = TOOLS,
) -> ModelDecision:
    if prompt == "read README.md":
        return ModelDecision(
            message="I should inspect the README before explaining the project.",
            tool_call=ToolCall(
                name="read_file",
                arguments={"path": "README.md"},
            ),
        )

    if prompt == "run tests":
        return ModelDecision(
            message="I should run the test suite before reporting status.",
            tool_call=ToolCall(
                name="run_tests",
                arguments={"command": "pytest"},
            ),
        )

    return ModelDecision(
        message="I can answer directly for this first toy version.",
        tool_call=None,
    )
```

This function is intentionally fake. Its job is to stand in for the LLM while
you build the runtime around it.

The architecture you are learning is:

```text
model returns structured intent
runtime decides whether that intent is allowed
```

## Step 6: Validate Tool Calls

The model should not be blindly trusted.

Add a small validator:

```python
def validate_tool_call(
    tool_call: ToolCall | None,
    tools: tuple[ToolSpec, ...] = TOOLS,
) -> str | None:
    if tool_call is None:
        return None

    allowed_names = {tool.name for tool in tools}
    if tool_call.name not in allowed_names:
        return f"unknown tool: {tool_call.name}"

    if tool_call.name == "read_file" and "path" not in tool_call.arguments:
        return "read_file requires a path"

    if tool_call.name == "run_tests" and "command" not in tool_call.arguments:
        return "run_tests requires a command"

    return None
```

This is a key coding-agent rule:

```text
the model can request a tool, but the runtime decides whether the call is valid
```

Later, validation will include permissions, sandbox rules, path checks, and
human approval.

## Step 7: Run One Agent Turn

Now wrap the model decision and validation in a turn loop:

```python
class CodingAgent:
    def __init__(self) -> None:
        self.transcript = Transcript()

    def run_turn(self, prompt: str) -> TurnResult:
        self.transcript.append("user", prompt)
        decision = mock_model_decide_next_action(prompt)
        error = validate_tool_call(decision.tool_call)

        if error:
            message = f"Blocked invalid tool call: {error}"
            stop_reason = "blocked"
            tool_call = decision.tool_call
        elif decision.tool_call:
            message = f"Model requested tool: {decision.tool_call.name}"
            stop_reason = "tool_requested"
            tool_call = decision.tool_call
        else:
            message = decision.message
            stop_reason = "completed"
            tool_call = None

        self.transcript.append("agent", message)
        return TurnResult(
            prompt=prompt,
            message=message,
            tool_call=tool_call,
            stop_reason=stop_reason,
        )
```

This is the first real agent shape:

```text
receive prompt
ask model for next action
validate model decision
record result
write transcript
return turn result
```

No file has been read yet. No shell command has run yet. That is good. The
runtime skeleton should exist before tools do real work.

## Step 8: Try Three Prompts

Use the same agent for multiple turns:

```python
agent = CodingAgent()

print(agent.run_turn("read README.md"))
print(agent.run_turn("run tests"))
print(agent.run_turn("explain what you can do"))
print(agent.transcript.replay())
```

Expected behavior:

```text
"read README.md" requests read_file with {"path": "README.md"}
"run tests" requests run_tests with {"command": "pytest"}
"explain what you can do" answers directly
transcript contains all three user turns and agent messages
```

This is still a toy. But it teaches the right mental model:

```text
tools are selected by model decision, not by keyword routing in the runtime
```

## Step 9: Understand Stop Reasons

Every turn should say why it stopped.

For now, use:

```text
completed:
    the model answered without asking for a tool

tool_requested:
    the model requested a valid tool call

blocked:
    the model requested an invalid or disallowed tool call

error:
    something failed while running the turn
```

Stop reasons make the agent debuggable. Later, they decide whether the loop
executes a tool, asks for permission, runs tests, or reports a failure.

## Step 10: The Week 01 Build

Your build this week is one small file:

```text
claudecode/agent.py
```

It should contain:

```text
ToolSpec
ToolCall
ModelDecision
TurnResult
Transcript
TOOLS
mock_model_decide_next_action
validate_tool_call
CodingAgent
```

Keep the implementation short. The goal is not completeness. The goal is to
make the model-decision loop concrete.

## Step 11: Done Checklist

You are done when:

- `CodingAgent.run_turn` accepts a prompt
- the mock model can return a direct answer
- the mock model can return a `ToolCall`
- the runtime validates the requested tool name
- the turn returns a `TurnResult`
- the turn appends user and agent entries to the transcript
- the result includes a `stop_reason`
- you can explain the loop as `prompt -> model decision -> validation -> result -> transcript`

Stop there.

## Skool Submission

Use this format:

```text
Week 01 Submission

My agent loop in one sentence:

The objects I built:

Example prompt:

Model decision:

Runtime validation:

What the transcript stores:

One thing I want reviewed:
```
