# Week 13: Compact Context

The transcript grows every turn.

At first that is good:

```text
more transcript = more memory
```

But eventually it becomes a problem:

```text
too much transcript = too many tokens for the model request
```

Compaction solves that by replacing older messages with a summary while keeping
the recent messages exactly.

The shape is:

```text
old transcript:
    many old messages + recent messages

compacted transcript:
    system summary + recent messages
```

This week builds a small context compactor.

## Step 1: Understand What Compaction Does

Compaction is not deletion.

It is controlled compression.

The agent keeps:

```text
1. a summary of older work
2. the most recent messages verbatim
```

Why keep recent messages exactly?

```text
recent messages contain the current task edge
recent tool calls may still matter
recent code snippets may be needed for exact edits
```

Why summarize older messages?

```text
older messages are useful as background,
but they usually do not need to be replayed word for word
```

Example:

```text
Before compaction:
    user: build bash tool
    assistant: wrote bash design
    tool: tests passed
    user: build TodoWrite
    assistant: wrote todo lesson
    user: now clarify Week 11
    assistant: updated read-file windows

After compaction:
    system: summary of bash tool and TodoWrite work
    user: now clarify Week 11
    assistant: updated read-file windows
```

The model still knows the history, but the request is smaller.

## Step 2: Define The Config

Create:

```text
claudecode/compact.py
```

Start with config:

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class CompactionConfig:
    preserve_recent_messages: int = 4
    max_estimated_tokens: int = 10_000
```

Read it as:

```text
preserve_recent_messages:
    how many recent messages stay unchanged

max_estimated_tokens:
    when older compactable messages are too large, compact
```

This makes compaction predictable:

```text
do not compact constantly
compact only after the transcript crosses a budget
```

## Step 3: Estimate Token Size

You do not need a perfect tokenizer for this lesson.

Use a rough estimate:

```python
from claudecode.session import ContentBlock, ConversationMessage


def estimate_block_tokens(block: ContentBlock) -> int:
```

Text blocks:

```python
    if hasattr(block, "text"):
        return len(block.text) // 4 + 1
```

Tool use blocks:

```python
    if hasattr(block, "name") and hasattr(block, "input"):
        return (len(block.name) + len(block.input)) // 4 + 1
```

Tool result blocks:

```python
    if hasattr(block, "tool_name") and hasattr(block, "output"):
        return (len(block.tool_name) + len(block.output)) // 4 + 1

    return 1
```

Now estimate one message:

```python
def estimate_message_tokens(message: ConversationMessage) -> int:
    return sum(
        estimate_block_tokens(block)
        for block in message.blocks
    )
```

And a session:

```python
def estimate_session_tokens(messages: list[ConversationMessage]) -> int:
    return sum(estimate_message_tokens(message) for message in messages)
```

This is good enough to decide when to compact.

## Step 4: Decide Whether To Compact

Do not compact if there are only a few messages.

```python
def should_compact(
    messages: list[ConversationMessage],
    config: CompactionConfig,
) -> bool:
    if len(messages) <= config.preserve_recent_messages:
        return False
```

Now check the token budget:

```python
    compactable = messages[:-config.preserve_recent_messages]
    return (
        estimate_session_tokens(compactable)
        >= config.max_estimated_tokens
    )
```

This means:

```text
only older messages count toward the compaction trigger
recent preserved messages are not the thing being summarized
```

## Step 5: Split Old And Recent Messages

When compaction happens, split the transcript.

```python
def split_for_compaction(
    messages: list[ConversationMessage],
    preserve_recent_messages: int,
) -> tuple[list[ConversationMessage], list[ConversationMessage]]:
    keep_from = max(0, len(messages) - preserve_recent_messages)
    return messages[:keep_from], messages[keep_from:]
```

Read it as:

```text
removed:
    older messages to summarize

preserved:
    recent messages kept exactly
```

But there is one important trap.

## Step 6: Do Not Split Tool Pairs

A `tool_result` depends on the assistant `tool_use` before it.

Bad compacted transcript:

```text
system: summary
tool: result for tool-1
assistant: final response
```

That is broken because the `tool_result` appears without the matching
assistant `tool_use`.

So if the first preserved message is a tool result, move the boundary backward.

```python
from claudecode.session import MessageRole, ToolResultBlock, ToolUseBlock


def starts_with_tool_result(message: ConversationMessage) -> bool:
    return bool(message.blocks) and isinstance(
        message.blocks[0],
        ToolResultBlock,
    )
```

Check whether a message contains a tool use:

```python
def has_tool_use(message: ConversationMessage) -> bool:
    return any(
        isinstance(block, ToolUseBlock)
        for block in message.blocks
    )
```

Now adjust the split:

```python
def safe_keep_from(
    messages: list[ConversationMessage],
    raw_keep_from: int,
) -> int:
    keep_from = raw_keep_from

    while keep_from > 0 and starts_with_tool_result(messages[keep_from]):
        if has_tool_use(messages[keep_from - 1]):
            keep_from -= 1
            break

        keep_from -= 1

    return keep_from
```

This keeps pairs together:

```text
assistant tool_use
tool tool_result
```

The model API expects those to stay adjacent in the transcript.

## Step 7: Split Safely

Use the safe boundary.

```python
def split_safely_for_compaction(
    messages: list[ConversationMessage],
    preserve_recent_messages: int,
) -> tuple[list[ConversationMessage], list[ConversationMessage]]:
    raw_keep_from = max(0, len(messages) - preserve_recent_messages)
    keep_from = safe_keep_from(messages, raw_keep_from)
    return messages[:keep_from], messages[keep_from:]
```

This small function protects the transcript shape.

Compaction is not allowed to create invalid message order.

## Step 8: Summarize One Block

Summaries should be compact.

```python
def truncate(text: str, max_chars: int = 160) -> str:
    if len(text) <= max_chars:
        return text
    return text[:max_chars] + "..."
```

Summarize block types:

```python
def summarize_block(block: ContentBlock) -> str:
    if hasattr(block, "text"):
        return truncate(block.text)

    if hasattr(block, "name") and hasattr(block, "input"):
        return truncate(f"tool_use {block.name}({block.input})")

    if hasattr(block, "tool_name") and hasattr(block, "output"):
        label = "error " if block.is_error else ""
        return truncate(
            f"tool_result {block.tool_name}: {label}{block.output}"
        )

    return "<unknown block>"
```

This gives the summary enough detail to remember what happened.

## Step 9: Summarize Removed Messages

Build a structured summary.

```python
def role_name(message: ConversationMessage) -> str:
    return message.role.value
```

```python
def summarize_messages(messages: list[ConversationMessage]) -> str:
    lines = [
        "<summary>",
        "Conversation summary:",
        f"- Scope: {len(messages)} earlier messages compacted.",
    ]
```

Add a timeline:

```python
    lines.append("- Key timeline:")
    for message in messages:
        content = " | ".join(
            summarize_block(block)
            for block in message.blocks
        )
        lines.append(f"  - {role_name(message)}: {content}")
```

Close the summary:

```python
    lines.append("</summary>")
    return "\n".join(lines)
```

The summary is not trying to be beautiful.

It is trying to preserve enough state for continuation.

## Step 10: Build The Continuation Message

After compaction, the first message becomes a synthetic system message.

```python
COMPACT_PREAMBLE = (
    "This session is being continued from a previous conversation that ran "
    "out of context. The summary below covers the earlier portion of the "
    "conversation.\n\n"
)
```

Format it:

```python
def continuation_text(summary: str, recent_preserved: bool) -> str:
    text = COMPACT_PREAMBLE + summary

    if recent_preserved:
        text += "\n\nRecent messages are preserved verbatim."

    text += (
        "\nContinue the conversation from where it left off without "
        "asking the user to recap."
    )
    return text
```

This system message tells the model:

```text
the earlier transcript has been summarized
the recent tail is still exact
continue directly
```

## Step 11: Define The Result

Return both the summary and the compacted message list.

```python
@dataclass(frozen=True)
class CompactionResult:
    summary: str
    messages: list[ConversationMessage]
    removed_message_count: int
```

The caller needs `removed_message_count` for logging and UI feedback.

## Step 12: Compact Messages

Now combine the pieces.

```python
from claudecode.session import TextBlock, ConversationMessage, MessageRole


def compact_messages(
    messages: list[ConversationMessage],
    config: CompactionConfig,
) -> CompactionResult:
```

If compaction is not needed, return unchanged:

```python
    if not should_compact(messages, config):
        return CompactionResult(
            summary="",
            messages=messages,
            removed_message_count=0,
        )
```

Split:

```python
    removed, preserved = split_safely_for_compaction(
        messages,
        config.preserve_recent_messages,
    )
```

Summarize removed messages:

```python
    summary = summarize_messages(removed)
```

Create the continuation system message:

```python
    system_message = ConversationMessage(
        role=MessageRole.SYSTEM,
        blocks=[
            TextBlock(
                continuation_text(
                    summary,
                    recent_preserved=bool(preserved),
                )
            )
        ],
    )
```

Return compacted messages:

```python
    return CompactionResult(
        summary=summary,
        messages=[system_message, *preserved],
        removed_message_count=len(removed),
    )
```

The compacted transcript is now:

```text
system summary
recent preserved message
recent preserved message
...
```

## Step 13: Compact A Session

Week 12 built `Session`.

Now add a helper that updates it:

```python
from claudecode.session import Session


def compact_session(
    session: Session,
    config: CompactionConfig,
) -> CompactionResult:
    result = compact_messages(session.messages, config)
```

Only update if messages were removed:

```python
    if result.removed_message_count:
        session.messages = result.messages

    return result
```

In a bigger runtime, also record compaction metadata:

```text
how many times compaction happened
how many messages were removed
the summary text
```

## Step 14: Auto Compact After A Turn

Do not compact in the middle of tool execution.

Compact after a turn finishes.

```python
def maybe_auto_compact(
    session: Session,
    input_tokens: int,
    threshold: int,
) -> CompactionResult | None:
    if input_tokens < threshold:
        return None
```

If threshold is crossed, compact aggressively:

```python
    result = compact_session(
        session,
        CompactionConfig(max_estimated_tokens=0),
    )

    if result.removed_message_count == 0:
        return None

    return result
```

The runtime can call this after:

```text
assistant answer
tool results
turn summary
```

That way the transcript is stable during the active turn.

## Step 15: Add Tests

Create:

```text
tests/test_compact.py
```

Test no compaction below budget:

```python
from claudecode.compact import (
    CompactionConfig,
    compact_messages,
    should_compact,
)
from claudecode.session import (
    ConversationMessage,
    MessageRole,
    TextBlock,
    ToolResultBlock,
    ToolUseBlock,
    user_text,
)
```

```python
def test_should_not_compact_short_session():
    messages = [user_text("hello")]

    assert should_compact(
        messages,
        CompactionConfig(
            preserve_recent_messages=4,
            max_estimated_tokens=1,
        ),
    ) is False
```

Test compaction preserves recent messages:

```python
def test_compact_messages_preserves_recent_tail():
    messages = [
        user_text("old one " + "x" * 100),
        user_text("old two " + "x" * 100),
        user_text("recent one"),
        user_text("recent two"),
    ]

    result = compact_messages(
        messages,
        CompactionConfig(
            preserve_recent_messages=2,
            max_estimated_tokens=1,
        ),
    )

    assert result.removed_message_count == 2
    assert result.messages[0].role == MessageRole.SYSTEM
    assert result.messages[1] == messages[2]
    assert result.messages[2] == messages[3]
```

Test tool pairs stay together:

```python
def test_compaction_does_not_split_tool_use_and_result():
    tool_id = "tool-1"
    messages = [
        user_text("search"),
        ConversationMessage(
            role=MessageRole.ASSISTANT,
            blocks=[
                ToolUseBlock(
                    id=tool_id,
                    name="grep_search",
                    input='{"pattern":"TodoWrite"}',
                )
            ],
        ),
        ConversationMessage(
            role=MessageRole.TOOL,
            blocks=[
                ToolResultBlock(
                    tool_use_id=tool_id,
                    tool_name="grep_search",
                    output="todos.py:81",
                    is_error=False,
                )
            ],
        ),
        user_text("done"),
    ]

    result = compact_messages(
        messages,
        CompactionConfig(
            preserve_recent_messages=2,
            max_estimated_tokens=1,
        ),
    )

    preserved_roles = [message.role for message in result.messages]

    assert MessageRole.ASSISTANT in preserved_roles
    assert MessageRole.TOOL in preserved_roles
```

## Done Checklist

You are done when:

- token size can be estimated roughly
- compaction only runs after the budget is crossed
- recent messages are preserved verbatim
- older messages become a summary
- tool-use/tool-result pairs are not split
- compacted transcripts start with a system continuation message
- auto compaction runs after a turn, not in the middle of tool execution

## Skool Submission

Use this format:

```text
Week 13 Submission

My compaction rule in one sentence:

What gets summarized:

What gets preserved:

Why tool_use/tool_result pairs must stay together:

One thing I want reviewed:
```
