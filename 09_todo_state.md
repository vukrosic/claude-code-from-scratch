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

The model sends the whole list every time.

```python
@dataclass(frozen=True)
class TodoWriteInput:
    todos: list[TodoItem]
```

The runtime returns old and new state:

```python
@dataclass(frozen=True)
class TodoWriteOutput:
    old_todos: list[TodoItem]
    new_todos: list[TodoItem]
    verification_nudge_needed: bool | None = None
```

This makes the update reviewable:

```text
oldTodos:
    previous saved list

newTodos:
    list the model just requested
```

## Step 4: Validate The List

Reject empty lists:

```python
def validate_todos(todos: list[TodoItem]) -> None:
    if not todos:
        raise ValueError("todos must not be empty")
```

Reject blank content:

```python
    if any(not todo.content.strip() for todo in todos):
        raise ValueError("todo content must not be empty")
```

Reject blank active forms:

```python
    if any(not todo.active_form.strip() for todo in todos):
        raise ValueError("todo activeForm must not be empty")
```

Do not require exactly one `in_progress` item.

Multiple active todos are allowed because later agent flows can run parallel
work.

## Step 5: Choose The Store Path

Persist todos in the current workspace.

```python
from pathlib import Path
import os


def todo_store_path() -> Path:
    override = os.environ.get("CLAWD_TODO_STORE")
    if override:
        return Path(override)

    return Path.cwd() / ".clawd-todos.json"
```

The environment override makes tests easy. The default file keeps state local to
the repo.

## Step 6: Load Old Todos

Read the previous state if it exists.

```python
import json


def load_todos(path: Path) -> list[TodoItem]:
    if not path.exists():
        return []

    raw_items = json.loads(path.read_text(encoding="utf-8"))
```

Convert JSON objects back into typed todo items:

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

Notice the JSON field name:

```text
activeForm
```

Use camelCase at the tool boundary because that is what the model-facing schema
uses.

## Step 7: Save Todos

Convert one todo to JSON:

```python
def todo_to_json(todo: TodoItem) -> dict:
    return {
        "content": todo.content,
        "activeForm": todo.active_form,
        "status": todo.status.value,
    }
```

Save the list:

```python
def save_todos(path: Path, todos: list[TodoItem]) -> None:
    path.parent.mkdir(parents=True, exist_ok=True)
    payload = [todo_to_json(todo) for todo in todos]
    path.write_text(
        json.dumps(payload, indent=2),
        encoding="utf-8",
    )
```

Keep this boring. Todo state should be easy to inspect by opening the JSON file.

## Step 8: Clear Storage When Everything Is Done

If every todo is completed, save an empty list.

```python
def persisted_todos(todos: list[TodoItem]) -> list[TodoItem]:
    all_done = all(todo.status == TodoStatus.COMPLETED for todo in todos)
    return [] if all_done else todos
```

Why clear it?

```text
finished sessions should not reopen with stale completed work
```

The tool still returns `new_todos` so the transcript records what was completed.

## Step 9: Add A Verification Nudge

A useful tiny check:

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

This does not block the model. It only returns a hint:

```text
you completed several steps, but none looked like verification
```

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
TodoWrite replaces the whole todo list
```

It is not an append tool. The model sends the current full state.

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
