# Week 07: Feed Tool Results Back Into The Loop

The agent is not done when it runs a tool.

The important loop is:

```text
model returns tool_use
runtime executes tool
runtime appends tool_result
model sees tool_result
model decides the next step
```

This is how the model learns what happened after `read_file`, `edit_file`, or
`bash`.

## Step 1: Define Conversation Blocks

Create:

```text
claudecode/conversation_loop.py
```

Start with three block types:

```python
from dataclasses import dataclass
from typing import Literal


@dataclass(frozen=True)
class TextBlock:
    text: str


@dataclass(frozen=True)
class ToolUseBlock:
    id: str
    name: str
    input: dict


@dataclass(frozen=True)
class ToolResultBlock:
    tool_use_id: str
    tool_name: str
    output: str
    is_error: bool
```

The ids matter.

```text
tool_use.id:
    created by the model provider

tool_result.tool_use_id:
    points back to the exact tool_use that produced this result
```

Without that id link, the model cannot reliably match results to tool calls.

## Step 2: Define Messages

A message is a role plus blocks.

```python
ContentBlock = TextBlock | ToolUseBlock | ToolResultBlock


@dataclass(frozen=True)
class Message:
    role: Literal["user", "assistant", "tool"]
    blocks: list[ContentBlock]
```

Use helpers so the loop code stays readable:

```python
def user_text(text: str) -> Message:
    return Message(role="user", blocks=[TextBlock(text)])


def assistant_message(blocks: list[ContentBlock]) -> Message:
    return Message(role="assistant", blocks=blocks)


def tool_result_message(
    tool_use_id: str,
    tool_name: str,
    output: str,
    is_error: bool,
) -> Message:
    return Message(
        role="tool",
        blocks=[
            ToolResultBlock(
                tool_use_id=tool_use_id,
                tool_name=tool_name,
                output=output,
                is_error=is_error,
            )
        ],
    )
```

Many APIs send tool results using a user-like role internally. Keeping `tool` in
your app state is still useful because it makes the transcript clearer.

## Step 3: Extract Tool Uses

After each assistant message, find requested tools.

```python
def pending_tool_uses(message: Message) -> list[ToolUseBlock]:
    return [
        block
        for block in message.blocks
        if isinstance(block, ToolUseBlock)
    ]
```

This is the branch point:

```text
no tool_use blocks:
    the turn is finished

one or more tool_use blocks:
    execute them and continue the loop
```

## Step 4: Define The Tool Executor

The loop should not know how each tool works.

It only needs a small executor protocol:

```python
from typing import Protocol


class ToolExecutor(Protocol):
    def execute(self, tool_name: str, tool_input: dict) -> str:
        ...
```

This lets `conversation_loop.py` call tools without importing every tool module.

Later your executor can route to:

```text
read_file
edit_file
write_file
bash
```

## Step 5: Execute One Tool Use

Turn one `ToolUseBlock` into one `ToolResultBlock`.

```python
def execute_tool_use(
    tool_use: ToolUseBlock,
    executor: ToolExecutor,
) -> Message:
```

Run the tool:

```python
    try:
        output = executor.execute(tool_use.name, tool_use.input)
        is_error = False
    except Exception as error:
        output = str(error)
        is_error = True
```

Return a linked result:

```python
    return tool_result_message(
        tool_use_id=tool_use.id,
        tool_name=tool_use.name,
        output=output,
        is_error=is_error,
    )
```

Tool errors are not thrown away. They become tool results with `is_error=True`,
so the model can recover.

## Step 6: Define The Model Client

The loop also should not know the model provider.

```python
class ModelClient(Protocol):
    def stream(self, messages: list[Message]) -> Message:
        ...
```

For this lesson, `stream` returns one complete assistant message.

Inside a real client, streaming events are collected into blocks:

```text
text delta -> TextBlock
tool_use event -> ToolUseBlock
message stop -> assistant message is complete
```

The runtime should execute tools only after the assistant message is complete.

## Step 7: Build The Turn Loop

Now wire the pieces together.

```python
def run_turn(
    messages: list[Message],
    user_prompt: str,
    model: ModelClient,
    executor: ToolExecutor,
    *,
    max_iterations: int = 8,
) -> list[Message]:
```

Append the user's message:

```python
    transcript = [*messages, user_text(user_prompt)]
```

Loop until the model stops asking for tools:

```python
    for _ in range(max_iterations):
        assistant = model.stream(transcript)
        transcript.append(assistant)

        tool_uses = pending_tool_uses(assistant)
        if not tool_uses:
            return transcript
```

Execute each requested tool and append each result:

```python
        for tool_use in tool_uses:
            result = execute_tool_use(tool_use, executor)
            transcript.append(result)
```

If the model keeps calling tools forever, stop:

```python
    raise RuntimeError("tool loop exceeded max_iterations")
```

The full behavior is:

```text
append user message
ask model
append assistant message
if assistant used tools:
    execute tools
    append tool results
    ask model again
else:
    finish
```

## Step 8: Convert Messages For The API

When sending messages to the model API, convert your internal blocks.

```python
def to_api_message(message: Message) -> dict:
    role = "assistant" if message.role == "assistant" else "user"
```

Tool results usually go back as user-side content:

```python
    content = []
    for block in message.blocks:
        if isinstance(block, TextBlock):
            content.append({"type": "text", "text": block.text})
        elif isinstance(block, ToolUseBlock):
            content.append({
                "type": "tool_use",
                "id": block.id,
                "name": block.name,
                "input": block.input,
            })
        elif isinstance(block, ToolResultBlock):
            content.append({
                "type": "tool_result",
                "tool_use_id": block.tool_use_id,
                "content": [
                    {"type": "text", "text": block.output}
                ],
                "is_error": block.is_error,
            })

    return {"role": role, "content": content}
```

Notice the role line:

```python
role = "assistant" if message.role == "assistant" else "user"
```

That means internal `tool` messages are sent to the provider as user-side
`tool_result` content.

## Step 9: Test The Loop

Create:

```text
tests/test_conversation_loop.py
```

Use a fake model that asks for a tool, then answers after seeing the result.

```python
from claudecode.conversation_loop import (
    Message,
    TextBlock,
    ToolUseBlock,
    run_turn,
)


class FakeModel:
    def __init__(self):
        self.calls = 0

    def stream(self, messages: list[Message]) -> Message:
        self.calls += 1

        if self.calls == 1:
            return Message(
                role="assistant",
                blocks=[
                    ToolUseBlock(
                        id="tool-1",
                        name="read_file",
                        input={"path": "README.md"},
                    )
                ],
            )

        return Message(
            role="assistant",
            blocks=[TextBlock("README says: hello")],
        )
```

Fake executor:

```python
class FakeExecutor:
    def execute(self, tool_name: str, tool_input: dict) -> str:
        assert tool_name == "read_file"
        assert tool_input == {"path": "README.md"}
        return "hello"
```

Test the transcript:

```python
def test_run_turn_feeds_tool_result_back_to_model():
    transcript = run_turn(
        messages=[],
        user_prompt="What does the README say?",
        model=FakeModel(),
        executor=FakeExecutor(),
    )

    assert [message.role for message in transcript] == [
        "user",
        "assistant",
        "tool",
        "assistant",
    ]

    tool_result = transcript[2].blocks[0]
    assert tool_result.tool_use_id == "tool-1"
    assert tool_result.output == "hello"

    final = transcript[-1].blocks[0]
    assert final.text == "README says: hello"
```

That test proves the key behavior:

```text
the second model call happens after the tool result is in the transcript
```

## Done Checklist

You are done when:

- assistant messages can contain `ToolUseBlock`
- each tool use produces a linked `ToolResultBlock`
- tool errors become `is_error=True` results
- the loop calls the model again after tool results
- the loop stops when the assistant returns no tool uses
- the loop has a max-iteration guard
- internal `tool` messages convert to API `tool_result` content

## Skool Submission

Use this format:

```text
Week 07 Submission

The tool-result loop in one sentence:

Example transcript roles:

How tool_use_id links the result:

What happens when a tool errors:

One thing I want reviewed:
```
