# Week 12: Persist The Session Transcript

An agent needs memory across turns.

That memory is the session transcript.

The transcript is not just a chat log. It stores:

```text
user messages
assistant messages
tool uses requested by the assistant
tool results produced by the runtime
```

This is what lets the next model call know:

```text
what the user asked
what the assistant decided
which tools ran
what those tools returned
```

This week builds a small JSONL session store.

## Step 1: Understand The Transcript Shape

From earlier weeks, a normal tool turn looks like this:

```text
1. user:
       "Find where TodoWrite is implemented."

2. assistant:
       tool_use grep_search {"pattern": "TodoWrite"}

3. tool:
       tool_result grep_search "todos.py:81:class TodoWriteInput"

4. assistant:
       "TodoWrite starts in todos.py around line 81."
```

The transcript must preserve that order.

Why?

```text
If the tool_result is missing,
the next model call does not know what grep_search found.

If the tool_use id is missing,
the runtime cannot link a result to the tool call that caused it.
```

The transcript is the source of truth for the conversation.

## Step 2: Define Roles

Create:

```text
claudecode/session.py
```

Start with the roles:

```python
from enum import StrEnum


class MessageRole(StrEnum):
    SYSTEM = "system"
    USER = "user"
    ASSISTANT = "assistant"
    TOOL = "tool"
```

Read them as:

```text
system:
    durable instructions

user:
    user prompt

assistant:
    model response, including tool_use blocks

tool:
    runtime result from executing a tool
```

The `tool` role is internal app state. When sending to some model APIs, tool
results may be converted into user-side `tool_result` content, like Week 07.

## Step 3: Define Content Blocks

A message can contain different block types.

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class TextBlock:
    text: str
```

Assistant tool request:

```python
@dataclass(frozen=True)
class ToolUseBlock:
    id: str
    name: str
    input: str
```

Runtime tool result:

```python
@dataclass(frozen=True)
class ToolResultBlock:
    tool_use_id: str
    tool_name: str
    output: str
    is_error: bool
```

The link is:

```text
ToolUseBlock.id == ToolResultBlock.tool_use_id
```

Example:

```text
assistant:
    ToolUseBlock(id="tool-1", name="bash", input="echo hi")

tool:
    ToolResultBlock(tool_use_id="tool-1", tool_name="bash", output="hi", is_error=False)
```

That link lets the transcript say:

```text
this exact output came from that exact tool call
```

## Step 4: Define Conversation Messages

Now wrap blocks inside a message.

```python
ContentBlock = TextBlock | ToolUseBlock | ToolResultBlock


@dataclass(frozen=True)
class ConversationMessage:
    role: MessageRole
    blocks: list[ContentBlock]
```

Add helpers for common messages:

```python
def user_text(text: str) -> ConversationMessage:
    return ConversationMessage(
        role=MessageRole.USER,
        blocks=[TextBlock(text)],
    )
```

Assistant message:

```python
def assistant_message(blocks: list[ContentBlock]) -> ConversationMessage:
    return ConversationMessage(
        role=MessageRole.ASSISTANT,
        blocks=blocks,
    )
```

Tool result message:

```python
def tool_result_message(
    tool_use_id: str,
    tool_name: str,
    output: str,
    is_error: bool,
) -> ConversationMessage:
    return ConversationMessage(
        role=MessageRole.TOOL,
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

These helpers keep the runtime loop easy to read.

## Step 5: Create A Session

A session has an id and a list of messages.

```python
from pathlib import Path
import time
import uuid


@dataclass
class Session:
    session_id: str
    created_at_ms: int
    updated_at_ms: int
    messages: list[ConversationMessage]
    path: Path | None = None
```

Create a new session:

```python
def now_ms() -> int:
    return int(time.time() * 1000)


def new_session(path: Path | None = None) -> Session:
    now = now_ms()
    return Session(
        session_id=str(uuid.uuid4()),
        created_at_ms=now,
        updated_at_ms=now,
        messages=[],
        path=path,
    )
```

The `path` is optional.

```text
path=None:
    in-memory session only

path=Path(...):
    append messages to disk as JSONL
```

## Step 6: Convert Blocks To JSON

The transcript should be structured JSON, not plain text.

```python
def block_to_json(block: ContentBlock) -> dict:
    if isinstance(block, TextBlock):
        return {
            "type": "text",
            "text": block.text,
        }
```

Tool use:

```python
    if isinstance(block, ToolUseBlock):
        return {
            "type": "tool_use",
            "id": block.id,
            "name": block.name,
            "input": block.input,
        }
```

Tool result:

```python
    if isinstance(block, ToolResultBlock):
        return {
            "type": "tool_result",
            "tool_use_id": block.tool_use_id,
            "tool_name": block.tool_name,
            "output": block.output,
            "is_error": block.is_error,
        }

    raise TypeError(f"unsupported block: {block!r}")
```

This gives every block a `type` field so loading can reconstruct it later.

## Step 7: Convert Messages To JSON

Messages contain a role and blocks.

```python
def message_to_json(message: ConversationMessage) -> dict:
    return {
        "role": message.role.value,
        "blocks": [
            block_to_json(block)
            for block in message.blocks
        ],
    }
```

Example JSON:

```json
{
  "role": "assistant",
  "blocks": [
    {
      "type": "tool_use",
      "id": "tool-1",
      "name": "grep_search",
      "input": "{\"pattern\":\"TodoWrite\"}"
    }
  ]
}
```

That is much better than storing:

```text
assistant called grep_search somehow
```

The runtime can replay structured JSON. It cannot reliably replay vague prose.

## Step 8: Store JSONL Records

Use JSONL: one JSON object per line.

Why JSONL?

```text
append one message without rewriting the whole file
recover by replaying lines in order
inspect the transcript with normal text tools
```

A session file starts with metadata:

```python
def meta_record(session: Session) -> dict:
    return {
        "type": "session_meta",
        "session_id": session.session_id,
        "created_at_ms": session.created_at_ms,
        "updated_at_ms": session.updated_at_ms,
    }
```

Each message becomes a record:

```python
def message_record(message: ConversationMessage) -> dict:
    return {
        "type": "message",
        "message": message_to_json(message),
    }
```

The file looks like:

```jsonl
{"type":"session_meta","session_id":"abc","created_at_ms":1,"updated_at_ms":1}
{"type":"message","message":{"role":"user","blocks":[{"type":"text","text":"hello"}]}}
{"type":"message","message":{"role":"assistant","blocks":[{"type":"text","text":"hi"}]}}
```

That is the transcript on disk.

## Step 9: Save A Full Snapshot

Sometimes you need to write the whole session.

```python
import json


def render_jsonl_snapshot(session: Session) -> str:
    records = [meta_record(session)]
    records.extend(
        message_record(message)
        for message in session.messages
    )
    return "\n".join(json.dumps(record) for record in records) + "\n"
```

Save it:

```python
def save_session(session: Session, path: Path) -> None:
    path.parent.mkdir(parents=True, exist_ok=True)
    path.write_text(
        render_jsonl_snapshot(session),
        encoding="utf-8",
    )
```

This is useful when creating a new session file for the first time.

## Step 10: Append Messages

Most of the time, the runtime appends one message at a time.

```python
def append_message(session: Session, message: ConversationMessage) -> None:
    session.updated_at_ms = now_ms()
    session.messages.append(message)
```

If there is no path, stop here:

```python
    if session.path is None:
        return
```

If the file does not exist yet, save a full snapshot:

```python
    if not session.path.exists():
        save_session(session, session.path)
        return
```

Otherwise append only the new message:

```python
    with session.path.open("a", encoding="utf-8") as file:
        file.write(json.dumps(message_record(message)) + "\n")
```

This is the important persistence behavior:

```text
push message in memory
append same message to JSONL
```

If appending fails in a production runtime, roll back the in-memory append. For
this lesson, keep the code small.

## Step 11: Load Blocks From JSON

Now build the reverse path.

```python
def block_from_json(data: dict) -> ContentBlock:
    block_type = data["type"]
```

Text:

```python
    if block_type == "text":
        return TextBlock(text=data["text"])
```

Tool use:

```python
    if block_type == "tool_use":
        return ToolUseBlock(
            id=data["id"],
            name=data["name"],
            input=data["input"],
        )
```

Tool result:

```python
    if block_type == "tool_result":
        return ToolResultBlock(
            tool_use_id=data["tool_use_id"],
            tool_name=data["tool_name"],
            output=data["output"],
            is_error=data["is_error"],
        )

    raise ValueError(f"unsupported block type: {block_type}")
```

Loading must reject unknown block types. Silent transcript corruption is worse
than a clear error.

## Step 12: Load Messages From JSON

Convert one saved message back into app state.

```python
def message_from_json(data: dict) -> ConversationMessage:
    return ConversationMessage(
        role=MessageRole(data["role"]),
        blocks=[
            block_from_json(block)
            for block in data["blocks"]
        ],
    )
```

This reconstructs the same objects the runtime uses while running.

## Step 13: Load A Session From JSONL

Replay the file line by line.

```python
def load_session(path: Path) -> Session:
    session_id: str | None = None
    created_at_ms: int | None = None
    updated_at_ms: int | None = None
    messages: list[ConversationMessage] = []
```

Parse each record:

```python
    for raw_line in path.read_text(encoding="utf-8").splitlines():
        if not raw_line.strip():
            continue

        record = json.loads(raw_line)
        record_type = record["type"]
```

Handle metadata:

```python
        if record_type == "session_meta":
            session_id = record["session_id"]
            created_at_ms = record["created_at_ms"]
            updated_at_ms = record["updated_at_ms"]
```

Handle messages:

```python
        elif record_type == "message":
            messages.append(
                message_from_json(record["message"])
            )
```

Reject unknown record types:

```python
        else:
            raise ValueError(f"unsupported JSONL record type: {record_type}")
```

Return the restored session:

```python
    now = now_ms()
    return Session(
        session_id=session_id or str(uuid.uuid4()),
        created_at_ms=created_at_ms or now,
        updated_at_ms=updated_at_ms or created_at_ms or now,
        messages=messages,
        path=path,
    )
```

That is how resume works:

```text
read JSONL file
replay message records
rebuild session.messages
send those messages to the model on the next turn
```

## Step 14: Connect It To The Turn Loop

In Week 07, the loop appended messages like this:

```text
user message
assistant message
tool result message
assistant message
```

Now those appends should go through the session:

```python
append_message(session, user_text(user_prompt))
```

After the model returns:

```python
append_message(session, assistant_message(blocks))
```

After a tool runs:

```python
append_message(
    session,
    tool_result_message(
        tool_use_id="tool-1",
        tool_name="grep_search",
        output="todos.py:81:class TodoWriteInput",
        is_error=False,
    ),
)
```

The model does not remember because it is magic.

It remembers because the runtime sends the transcript back on the next request.

## Step 15: Add Tests

Create:

```text
tests/test_session.py
```

Test save and load:

```python
from claudecode.session import (
    ConversationMessage,
    MessageRole,
    TextBlock,
    ToolResultBlock,
    ToolUseBlock,
    append_message,
    load_session,
    new_session,
    save_session,
    tool_result_message,
    user_text,
)
```

```python
def test_session_persists_and_restores_jsonl(tmp_path):
    path = tmp_path / "session.jsonl"
    session = new_session()

    append_message(session, user_text("hello"))
    append_message(
        session,
        ConversationMessage(
            role=MessageRole.ASSISTANT,
            blocks=[
                TextBlock("thinking"),
                ToolUseBlock(
                    id="tool-1",
                    name="bash",
                    input="echo hi",
                ),
            ],
        ),
    )
    append_message(
        session,
        tool_result_message("tool-1", "bash", "hi", False),
    )

    save_session(session, path)
    restored = load_session(path)

    assert restored.session_id == session.session_id
    assert len(restored.messages) == 3
    assert restored.messages[0].role == MessageRole.USER
    assert restored.messages[1].role == MessageRole.ASSISTANT
    assert restored.messages[2].role == MessageRole.TOOL
```

Test append-to-disk:

```python
def test_append_message_writes_jsonl_record(tmp_path):
    path = tmp_path / "session.jsonl"
    session = new_session(path=path)

    append_message(session, user_text("hi"))
    append_message(
        session,
        ConversationMessage(
            role=MessageRole.ASSISTANT,
            blocks=[TextBlock("hello")],
        ),
    )

    restored = load_session(path)

    assert len(restored.messages) == 2
    assert restored.messages[0] == user_text("hi")
```

Test tool linking:

```python
def test_tool_result_keeps_tool_use_id():
    result = tool_result_message(
        tool_use_id="tool-123",
        tool_name="grep_search",
        output="match",
        is_error=False,
    )

    block = result.blocks[0]

    assert isinstance(block, ToolResultBlock)
    assert block.tool_use_id == "tool-123"
```

## Done Checklist

You are done when:

- roles include `system`, `user`, `assistant`, and `tool`
- blocks include `text`, `tool_use`, and `tool_result`
- tool results keep the matching `tool_use_id`
- a session has `session_id`, timestamps, and messages
- messages can be saved as JSONL records
- sessions can be loaded by replaying JSONL
- the turn loop appends user, assistant, and tool messages to the session

## Skool Submission

Use this format:

```text
Week 12 Submission

My transcript rule in one sentence:

Example message order:

How tool_use links to tool_result:

Why JSONL is useful:

One thing I want reviewed:
```
