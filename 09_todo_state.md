# Week 09: Track Todo State

Long coding turns need visible state.

The simple pattern is:

```text
model sends full todo list
runtime validates it
runtime saves it
runtime returns oldTodos and newTodos
```

This week builds a `TodoWrite` tool.

## Step 1: Define Todo Status

Create:

```text
claudecode/todos.py
```

Start with the only statuses the tool accepts:

```python
from dataclasses import dataclass
from enum import StrEnum


class TodoStatus(StrEnum):
    PENDING = "pending"
    IN_PROGRESS = "in_progress"
    COMPLETED = "completed"
```

These are state labels, not prose. Keep them stable because the model and UI
will both depend on them.

## Step 2: Define A Todo Item

Each todo has normal text plus an active form.

```python
@dataclass(frozen=True)
class TodoItem:
    content: str
    active_form: str
    status: TodoStatus
```

Read it as:

```text
content:
    "Run tests"

active_form:
    "Running tests"

status:
    pending, in_progress, or completed
```

Why have `active_form`?

```text
content describes the task
active_form describes what is happening while it is active
```

That gives the UI a nicer live label without making the model rewrite the task
text.

## Step 3: Define Tool Input And Output

The model only sends the new full list.

```python
@dataclass(frozen=True)
class TodoWriteInput:
    todos: list[TodoItem]
```

Why the whole list?

```text
append/update/delete APIs need ids or merge rules
full replacement is simpler:
    "this is the current todo state now"
```

Now define the runtime output:

```python
@dataclass(frozen=True)
class TodoWriteOutput:
    old_todos: list[TodoItem]
    new_todos: list[TodoItem]
    verification_nudge_needed: bool | None = None
```

This is the part that can feel confusing:

```text
oldTodos:
    loaded by the runtime before writing
    not sent by the model

newTodos:
    the full list the model just requested
```

You need `oldTodos` so the runtime can show what changed.

For example:

```text
oldTodos:
    Add tool: in_progress
    Run tests: pending

newTodos:
    Add tool: completed
    Run tests: in_progress
```

The model does not need to calculate that diff. The runtime can report it.

## Step 4: Validate The List

Validation is intentionally small.

```python
def validate_todos(todos: list[TodoItem]) -> None:
    if not todos:
        raise ValueError("todos must not be empty")

    if any(not todo.content.strip() for todo in todos):
        raise ValueError("todo content must not be empty")

    if any(not todo.active_form.strip() for todo in todos):
        raise ValueError("todo activeForm must not be empty")
```

Do not require exactly one `in_progress` item.

Multiple active todos are allowed because later agent flows can run parallel
work.

## Step 5: Choose The Store Path

Persist todos in one JSON file.

```python
from pathlib import Path
import os


def todo_store_path() -> Path:
    override = os.environ.get("CLAWD_TODO_STORE")
    if override:
        return Path(override)

    return Path.cwd() / ".clawd-todos.json"
```

The default file is local to the workspace. The environment override is only so
tests can write to a temp file.

## Step 6: Load Old Todos

Before writing the new list, load the previous saved list.

```python
import json


def load_todos(path: Path) -> list[TodoItem]:
    if not path.exists():
        return []

    raw_items = json.loads(path.read_text(encoding="utf-8"))
```

```python
    return [
        TodoItem(
            content=item["content"],
            active_form=item["activeForm"],
            status=TodoStatus(item["status"]),
        )
        for item in raw_items
    ]
```

This is where `oldTodos` comes from:

```text
oldTodos = load_todos(path)
```

It is not part of the model input.

## Step 7: Save Todos

Save the new list using model-facing field names.

```python
def todo_to_json(todo: TodoItem) -> dict:
    return {
        "content": todo.content,
        "activeForm": todo.active_form,
        "status": todo.status.value,
    }
```

```python
def save_todos(path: Path, todos: list[TodoItem]) -> None:
    path.parent.mkdir(parents=True, exist_ok=True)
    payload = [todo_to_json(todo) for todo in todos]
    path.write_text(
        json.dumps(payload, indent=2),
        encoding="utf-8",
    )
```

The stored JSON uses `activeForm`, not `active_form`, because that is the tool
schema field name.

## Step 8: Clear Storage When Everything Is Done

If every todo is completed, persist an empty list.

```python
def persisted_todos(todos: list[TodoItem]) -> list[TodoItem]:
    all_done = all(todo.status == TodoStatus.COMPLETED for todo in todos)
    return [] if all_done else todos
```

The output still returns the completed `newTodos`.

Only the stored file is cleared.

```text
newTodos:
    shows what was completed in this tool result

.clawd-todos.json:
    becomes []
```

That prevents the next session from reopening stale finished work.

## Step 9: Add A Verification Nudge

The reference adds one tiny hint.

```python
def needs_verification_nudge(todos: list[TodoItem]) -> bool | None:
    all_done = all(todo.status == TodoStatus.COMPLETED for todo in todos)
    has_many_steps = len(todos) >= 3
    mentions_verification = any(
        "verif" in todo.content.lower()
        for todo in todos
    )

    return True if all_done and has_many_steps and not mentions_verification else None
```

Read it as:

```text
if everything is completed
and there were at least 3 todos
and none mention verification
then return True
```

This does not block anything. It is just a nudge that maybe tests or verification
were skipped.

## Step 10: Implement `todo_write`

Now combine the pieces.

```python
def todo_write(input: TodoWriteInput) -> TodoWriteOutput:
    validate_todos(input.todos)

    path = todo_store_path()
    old_todos = load_todos(path)

    save_todos(path, persisted_todos(input.todos))

    return TodoWriteOutput(
        old_todos=old_todos,
        new_todos=input.todos,
        verification_nudge_needed=needs_verification_nudge(input.todos),
    )
```

The key behavior:

```text
1. validate the new list
2. load the old saved list
3. save the new list, or [] if all done
4. return oldTodos and newTodos
```

That is why `oldTodos` exists: it lets the runtime say "before this call, the
todo state was X; after this call, the model requested Y."

## Step 11: Parse Tool Arguments

The model sends JSON-like arguments.

```python
def todo_from_json(item: dict) -> TodoItem:
    return TodoItem(
        content=item["content"],
        active_form=item["activeForm"],
        status=TodoStatus(item["status"]),
    )
```

Build the tool wrapper:

```python
def todo_write_tool(arguments: dict) -> dict:
    input = TodoWriteInput(
        todos=[
            todo_from_json(item)
            for item in arguments["todos"]
        ]
    )

    output = todo_write(input)
```

Return model-friendly field names:

```python
    return {
        "oldTodos": [todo_to_json(todo) for todo in output.old_todos],
        "newTodos": [todo_to_json(todo) for todo in output.new_todos],
        "verificationNudgeNeeded": output.verification_nudge_needed,
    }
```

## Step 12: Add The Tool Spec

The model-facing tool is simple:

```python
TODO_WRITE_TOOL = {
    "name": "TodoWrite",
    "description": "Update the structured task list for the current session.",
    "parameters": {
        "todos": "List of todo objects with content, activeForm, and status.",
    },
}
```

Each todo must have:

```text
content
activeForm
status: pending | in_progress | completed
```

Permission-wise, this is a workspace write because it persists a repo-local JSON
file.

## Step 13: Add Tests

Create:

```text
tests/test_todos.py
```

Test first write:

```python
import os

from claudecode.todos import (
    TodoItem,
    TodoStatus,
    TodoWriteInput,
    todo_write,
)


def test_todo_write_persists_and_returns_old_state(tmp_path):
    os.environ["CLAWD_TODO_STORE"] = str(tmp_path / "todos.json")

    first = todo_write(
        TodoWriteInput(
            todos=[
                TodoItem("Add tool", "Adding tool", TodoStatus.IN_PROGRESS),
                TodoItem("Run tests", "Running tests", TodoStatus.PENDING),
            ]
        )
    )

    assert first.old_todos == []
    assert len(first.new_todos) == 2
```

Test second write sees old todos:

```python
    second = todo_write(
        TodoWriteInput(
            todos=[
                TodoItem("Add tool", "Adding tool", TodoStatus.COMPLETED),
                TodoItem("Run tests", "Running tests", TodoStatus.COMPLETED),
                TodoItem("Verify", "Verifying", TodoStatus.COMPLETED),
            ]
        )
    )

    assert len(second.old_todos) == 2
    assert len(second.new_todos) == 3
```

Clean up the env override:

```python
    os.environ.pop("CLAWD_TODO_STORE", None)
```

Test invalid payloads:

```python
def test_todo_write_rejects_blank_content():
    try:
        todo_write(
            TodoWriteInput(
                todos=[
                    TodoItem("   ", "Doing it", TodoStatus.PENDING)
                ]
            )
        )
    except ValueError as error:
        assert "todo content must not be empty" in str(error)
    else:
        raise AssertionError("expected blank todo content to fail")
```

Test verification nudge:

```python
def test_todo_write_sets_verification_nudge():
    output = todo_write(
        TodoWriteInput(
            todos=[
                TodoItem("Write tests", "Writing tests", TodoStatus.COMPLETED),
                TodoItem("Fix errors", "Fixing errors", TodoStatus.COMPLETED),
                TodoItem("Ship branch", "Shipping branch", TodoStatus.COMPLETED),
            ]
        )
    )

    assert output.verification_nudge_needed is True
```

## Done Checklist

You are done when:

- todo statuses are exactly `pending`, `in_progress`, and `completed`
- each todo has `content`, `activeForm`, and `status`
- `TodoWrite` replaces the full list
- old todo state is loaded before writing new state
- completed-all lists clear persisted state
- blank todo lists, content, and active forms are rejected
- the output returns `oldTodos`, `newTodos`, and `verificationNudgeNeeded`

## Skool Submission

Use this format:

```text
Week 09 Submission

My TodoWrite tool in one sentence:

Example todo list:

What oldTodos/newTodos show:

When persisted state is cleared:

One thing I want reviewed:
```
