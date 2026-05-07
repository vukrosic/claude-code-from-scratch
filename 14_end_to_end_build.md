# Week 14: Build The End-To-End Agent Runtime

This week is where the pieces become one agent.

So far you built parts:

```text
session transcript
tool result loop
permissions
file tools
bash tool
search tools
todo state
testing
compaction
```

The new concept is orchestration.

The runtime is the conductor:

```text
user prompt
-> append to session
-> ask model
-> append assistant message
-> execute requested tools
-> append tool results
-> repeat until final answer
-> compact if needed
```

The model decides what it wants to do.

The runtime owns what actually happens.

## Step 1: Define The Runtime Job

Create:

```text
claudecode/runtime.py
```

The runtime coordinates four things:

```text
session:
    transcript memory

model client:
    asks the LLM for the next assistant message

tool executor:
    runs local tools

permission policy:
    decides whether a tool is allowed
```

The runtime should not contain all tool code directly.

It should call smaller modules you already built.

## Step 2: Define Model Events

The model response can arrive as events.

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class TextDelta:
    text: str
```

Tool request:

```python
@dataclass(frozen=True)
class ToolUseEvent:
    id: str
    name: str
    input: str
```

Stop marker:

```python
@dataclass(frozen=True)
class MessageStop:
    pass
```

Use one type alias:

```python
AssistantEvent = TextDelta | ToolUseEvent | MessageStop
```

Why events?

```text
text can stream in chunks
tool input can arrive as a structured block
message_stop tells the runtime the assistant message is complete
```

The runtime should not execute tools until the assistant message is complete.

## Step 3: Define The Model Client

The model client gets the current transcript and returns assistant events.

```python
from typing import Protocol

from claudecode.session import ConversationMessage


@dataclass(frozen=True)
class ModelRequest:
    system_prompt: list[str]
    messages: list[ConversationMessage]
```

```python
class ModelClient(Protocol):
    def stream(self, request: ModelRequest) -> list[AssistantEvent]:
        ...
```

Read it as:

```text
runtime sends:
    system prompt + session messages

model returns:
    assistant text/tool_use events
```

This keeps provider code outside the runtime.

## Step 4: Build An Assistant Message From Events

Import the session block types:

```python
from claudecode.session import (
    ConversationMessage,
    MessageRole,
    TextBlock,
    ToolUseBlock,
)
```

Collect streamed text until you see a tool use or stop.

```python
def flush_text(text_parts: list[str], blocks: list) -> None:
    if text_parts:
        blocks.append(TextBlock("".join(text_parts)))
        text_parts.clear()
```

Now build the assistant message:

```python
def build_assistant_message(events: list[AssistantEvent]) -> ConversationMessage:
    text_parts: list[str] = []
    blocks = []
    finished = False
```

Handle text:

```python
    for event in events:
        if isinstance(event, TextDelta):
            text_parts.append(event.text)
```

Handle tool use:

```python
        elif isinstance(event, ToolUseEvent):
            flush_text(text_parts, blocks)
            blocks.append(
                ToolUseBlock(
                    id=event.id,
                    name=event.name,
                    input=event.input,
                )
            )
```

Handle stop:

```python
        elif isinstance(event, MessageStop):
            finished = True
```

Finish:

```python
    flush_text(text_parts, blocks)

    if not finished:
        raise RuntimeError("assistant stream ended without MessageStop")

    if not blocks:
        raise RuntimeError("assistant produced no content")

    return ConversationMessage(
        role=MessageRole.ASSISTANT,
        blocks=blocks,
    )
```

This function turns provider events into your own transcript format.

## Step 5: Extract Pending Tool Uses

After the assistant message is complete, find tool calls.

```python
from claudecode.session import ToolUseBlock


def pending_tool_uses(message: ConversationMessage) -> list[ToolUseBlock]:
    return [
        block
        for block in message.blocks
        if isinstance(block, ToolUseBlock)
    ]
```

This is the runtime branch:

```text
no tool uses:
    the turn is done

some tool uses:
    execute them and ask the model again
```

## Step 6: Define Tool Execution

The runtime calls an executor.

```python
class ToolExecutor(Protocol):
    def execute(self, tool_name: str, tool_input: str) -> str:
        ...
```

The executor hides tool routing.

For example:

```text
tool_name="grep_search"
tool_input='{"pattern":"TodoWrite"}'

executor:
    parse JSON
    call grep_search(...)
    return JSON/text result
```

The runtime should not care whether the tool is search, bash, or file editing.

## Step 7: Define Permission Checks

Use the permission layer from Week 05.

For this lesson, assume this protocol:

```python
class PermissionPolicy(Protocol):
    def allow(self, tool_name: str) -> bool:
        ...
```

The runtime checks before executing:

```text
model requested tool
runtime checks permission
if allowed:
    execute tool
else:
    append denied tool_result
```

Denied tools still become tool results.

Why?

```text
the model needs to know the tool did not run
```

## Step 8: Define Turn Summary

Return useful information after each user turn.

```python
@dataclass(frozen=True)
class TurnSummary:
    assistant_messages: list[ConversationMessage]
    tool_results: list[ConversationMessage]
    iterations: int
    auto_compacted: bool = False
```

This is not the final answer text.

It is runtime metadata:

```text
how many model iterations happened
how many tools ran
whether compaction happened
```

## Step 9: Define The Runtime

Now create the runtime object.

```python
from claudecode.session import Session, append_message, tool_result_message, user_text


@dataclass
class AgentRuntime:
    session: Session
    model: ModelClient
    tools: ToolExecutor
    permissions: PermissionPolicy
    system_prompt: list[str]
    max_iterations: int = 8
```

The runtime owns the loop.

The model does not get to execute tools by itself.

## Step 10: Start A Turn

A user turn begins by appending the user message.

```python
def run_turn(runtime: AgentRuntime, user_prompt: str) -> TurnSummary:
    append_message(runtime.session, user_text(user_prompt))

    assistant_messages: list[ConversationMessage] = []
    tool_results: list[ConversationMessage] = []
```

Why append before calling the model?

```text
the model request should include the user prompt
the session should persist what the user asked
```

The transcript is always updated in order.

## Step 11: Ask The Model

Inside the loop, build a model request from current session state.

```python
    for iteration in range(1, runtime.max_iterations + 1):
        request = ModelRequest(
            system_prompt=runtime.system_prompt,
            messages=runtime.session.messages,
        )

        events = runtime.model.stream(request)
        assistant = build_assistant_message(events)
```

Append the assistant message:

```python
        append_message(runtime.session, assistant)
        assistant_messages.append(assistant)
```

At this point, the transcript contains:

```text
user message
assistant message
```

If the assistant asked for tools, those tool-use blocks are inside the assistant
message.

## Step 12: Stop If No Tools Were Requested

Check pending tool uses:

```python
        tool_uses = pending_tool_uses(assistant)
        if not tool_uses:
            return TurnSummary(
                assistant_messages=assistant_messages,
                tool_results=tool_results,
                iterations=iteration,
            )
```

This is a final assistant answer.

The loop stops because the model did not request more runtime work.

## Step 13: Execute Requested Tools

If there are tool uses, execute each one.

```python
        for tool_use in tool_uses:
            if not runtime.permissions.allow(tool_use.name):
                result = tool_result_message(
                    tool_use_id=tool_use.id,
                    tool_name=tool_use.name,
                    output=f"Permission denied for tool: {tool_use.name}",
                    is_error=True,
                )
```

Allowed tool:

```python
            else:
                try:
                    output = runtime.tools.execute(
                        tool_use.name,
                        tool_use.input,
                    )
                    result = tool_result_message(
                        tool_use.id,
                        tool_use.name,
                        output,
                        False,
                    )
                except Exception as error:
                    result = tool_result_message(
                        tool_use.id,
                        tool_use.name,
                        str(error),
                        True,
                    )
```

Append the result:

```python
            append_message(runtime.session, result)
            tool_results.append(result)
```

Now the transcript contains:

```text
assistant tool_use
tool tool_result
```

The next loop iteration sends both back to the model.

## Step 14: Continue After Tool Results

After tool results are appended, the loop does not return yet.

It goes back to:

```text
ask model again
```

That second model call sees:

```text
user prompt
assistant tool request
tool result
```

Then the model can answer:

```text
"I found TodoWrite in claudecode/todos.py."
```

This is the heart of a coding agent:

```text
the model can act,
but only through runtime tools,
and the runtime records every action.
```

## Step 15: Guard Against Infinite Loops

If the model keeps calling tools forever, stop.

```python
    raise RuntimeError("conversation loop exceeded max_iterations")
```

Without this guard, a bad model response can create an endless tool loop.

Use a small value while learning:

```text
max_iterations=8
```

That is enough for many simple coding turns.

## Step 16: Add Compaction After The Turn

Week 13 built compaction.

In a full runtime, call it after the turn finishes.

Simple version:

```python
from claudecode.compact import CompactionConfig, compact_session, estimate_session_tokens
```

```python
def maybe_compact_after_turn(runtime: AgentRuntime) -> bool:
    tokens = estimate_session_tokens(runtime.session.messages)
    if tokens < 100_000:
        return False

    result = compact_session(
        runtime.session,
        CompactionConfig(max_estimated_tokens=0),
    )
    return result.removed_message_count > 0
```

Why after the turn?

```text
do not compact between assistant tool_use and tool tool_result
do not rewrite the transcript while tools are still running
compact only once the turn is stable
```

## Step 17: Wire A Tool Registry

Now build a small executor.

```python
import json


class ToolRegistry:
    def __init__(self):
        self._tools = {}

    def register(self, name: str, function):
        self._tools[name] = function
```

Execute:

```python
    def execute(self, tool_name: str, tool_input: str) -> str:
        if tool_name not in self._tools:
            raise ValueError(f"unknown tool: {tool_name}")

        arguments = json.loads(tool_input or "{}")
        output = self._tools[tool_name](arguments)
        return json.dumps(output)
```

Register previous tools:

```python
registry = ToolRegistry()
registry.register("glob_search", glob_search_tool)
registry.register("grep_search", grep_search_tool)
registry.register("read_file", read_file_tool)
registry.register("edit_file", edit_file_tool)
registry.register("bash", bash_tool)
registry.register("TodoWrite", todo_write_tool)
registry.register("run_tests", run_tests_tool)
```

The runtime now has one executor interface for every local tool.

## Step 18: Add A Minimal Permission Policy

Start simple.

```python
class SimplePermissionPolicy:
    def __init__(self, allowed_tools: set[str]):
        self.allowed_tools = allowed_tools

    def allow(self, tool_name: str) -> bool:
        return tool_name in self.allowed_tools
```

Example:

```python
permissions = SimplePermissionPolicy(
    allowed_tools={
        "glob_search",
        "grep_search",
        "read_file",
        "TodoWrite",
    }
)
```

This keeps dangerous tools like `bash` and `edit_file` out until the runtime mode
allows them.

## Step 19: Build A Runtime Factory

Create one helper that assembles the runtime.

```python
def build_runtime(
    model: ModelClient,
    session: Session,
    tools: ToolExecutor,
    permissions: PermissionPolicy,
) -> AgentRuntime:
    return AgentRuntime(
        session=session,
        model=model,
        tools=tools,
        permissions=permissions,
        system_prompt=[
            "You are a coding agent.",
            "Use tools when you need repo context.",
            "Do not claim a tool result unless it appears in the transcript.",
        ],
    )
```

The system prompt is not a replacement for runtime checks.

It is just guidance.

Permissions, file writes, and shell execution are still enforced by code.

## Step 20: Test The End-To-End Loop

Create:

```text
tests/test_runtime.py
```

Fake model:

```python
class FakeModel:
    def __init__(self):
        self.calls = 0

    def stream(self, request: ModelRequest) -> list[AssistantEvent]:
        self.calls += 1

        if self.calls == 1:
            return [
                ToolUseEvent(
                    id="tool-1",
                    name="grep_search",
                    input='{"pattern":"TodoWrite"}',
                ),
                MessageStop(),
            ]

        return [
            TextDelta("TodoWrite is in claudecode/todos.py."),
            MessageStop(),
        ]
```

Fake tools:

```python
class FakeTools:
    def execute(self, tool_name: str, tool_input: str) -> str:
        assert tool_name == "grep_search"
        assert tool_input == '{"pattern":"TodoWrite"}'
        return "claudecode/todos.py:81:class TodoWriteInput"
```

Allow everything in this test:

```python
class AllowAll:
    def allow(self, tool_name: str) -> bool:
        return True
```

Test:

```python
from claudecode.session import MessageRole, new_session
from claudecode.runtime import AgentRuntime, run_turn


def test_runtime_executes_tool_loop_end_to_end():
    runtime = AgentRuntime(
        session=new_session(),
        model=FakeModel(),
        tools=FakeTools(),
        permissions=AllowAll(),
        system_prompt=["system"],
    )

    summary = run_turn(runtime, "Find TodoWrite")

    assert summary.iterations == 2
    assert len(summary.tool_results) == 1
    assert [message.role for message in runtime.session.messages] == [
        MessageRole.USER,
        MessageRole.ASSISTANT,
        MessageRole.TOOL,
        MessageRole.ASSISTANT,
    ]
```

That test proves the whole spine:

```text
user -> assistant tool_use -> tool_result -> assistant final answer
```

## Step 21: Test Permission Denial

The runtime should record denied tools as tool results.

```python
class DenyAll:
    def allow(self, tool_name: str) -> bool:
        return False
```

Test:

```python
def test_runtime_records_denied_tool_as_error_result():
    runtime = AgentRuntime(
        session=new_session(),
        model=FakeModel(),
        tools=FakeTools(),
        permissions=DenyAll(),
        system_prompt=["system"],
        max_iterations=1,
    )

    try:
        run_turn(runtime, "Find TodoWrite")
    except RuntimeError:
        pass

    tool_message = runtime.session.messages[2]
    tool_block = tool_message.blocks[0]

    assert tool_block.is_error is True
    assert "Permission denied" in tool_block.output
```

The permission denial still enters the transcript, so the model can adapt on the
next iteration.

## Step 22: What You Built

At this point, the tiny agent has the same core shape as a real coding agent:

```text
Session:
    remembers the conversation

ModelClient:
    decides the next assistant message

ToolExecutor:
    runs local capabilities

PermissionPolicy:
    gates dangerous actions

AgentRuntime:
    coordinates the loop
```

The final architecture:

```text
run_turn(user_prompt)
    append user message
    loop:
        model.stream(session.messages)
        append assistant message
        if no tools:
            finish
        for each tool:
            check permission
            execute or deny
            append tool_result
    maybe compact
```

That is the foundation.

Everything else is improvement:

```text
better model provider
better tool schemas
better UI
better permission prompts
better compaction summaries
background workers
sub-agents
```

But the core loop is now complete.

## Done Checklist

You are done when:

- runtime appends user messages to the session
- model events become assistant messages
- assistant tool uses are extracted after the message completes
- permissions are checked before execution
- tool results are appended to the transcript
- the model is called again after tool results
- the loop stops when there are no tool uses
- max iterations prevents infinite loops
- compaction can run after a completed turn

## Skool Submission

Use this format:

```text
Week 14 Submission

My runtime loop in one sentence:

Example transcript roles:

Where permissions run:

Where tool results are appended:

One thing I want reviewed:
```
