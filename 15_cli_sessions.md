# Week 15: Add A CLI That Can Resume Sessions

This week you turn the runtime into something you can actually run from a
terminal.

The new concept is the session boundary.

A coding agent is not only:

```text
model + tools + runtime
```

It also needs a way to answer:

```text
which conversation am I continuing?
where is that conversation saved?
is this session allowed to run in this workspace?
```

That is what the CLI owns.

## Step 1: Know What The CLI Is Responsible For

The runtime from Week 14 handles one turn:

```text
user prompt -> model -> tools -> final assistant answer
```

The CLI handles everything around that turn:

```text
read command-line arguments
create or resume a session
build the runtime
send the prompt into the runtime
print the final assistant text
save the updated session
```

Keep those jobs separate.

The runtime should not parse command-line arguments.

The CLI should not manually run tools.

## Step 2: Store Sessions In A Managed Folder

Create:

```text
claudecode/session_store.py
```

A managed session is just a session file in a predictable location.

Use this folder shape:

```text
.claw/sessions/<workspace-fingerprint>/
```

Why a hidden folder?

Because sessions are project-local agent state. They are useful while working in
the repo, but they are not source code.

Why the workspace fingerprint?

Because two repos can both have a session named `session-alpha`. The fingerprint
keeps each repo's sessions in its own namespace.

Each session file should use JSONL:

```text
.claw/sessions/a1b2c3/session-abc123.jsonl
```

JSONL means one JSON object per line.

That makes appending new messages simple:

```text
line 1: session metadata
line 2: user message
line 3: assistant message
line 4: tool result
```

The CLI can append without rewriting one giant JSON document.

## Step 3: Define A Session Handle

The CLI does not need the full transcript when it is only choosing a session.

It first needs a small handle:

```python
from dataclasses import dataclass
from pathlib import Path


@dataclass(frozen=True)
class SessionHandle:
    id: str
    path: Path
```

Read this as:

```text
id:
    human-friendly session name

path:
    exact file on disk
```

For example:

```python
SessionHandle(
    id="session-abc123",
    path=Path(".claw/sessions/a1b2c3/session-abc123.jsonl"),
)
```

This tiny object is useful because the CLI can pass it around before loading the
whole conversation.

Add a helper for file paths too:

```python
def handle_from_path(path: Path) -> SessionHandle:
    return SessionHandle(id=path.stem, path=path)
```

## Step 4: Create A New Managed Session

Add:

```python
import hashlib
from pathlib import Path


def workspace_fingerprint(workspace: Path) -> str:
    digest = hashlib.sha256(str(workspace.resolve()).encode("utf-8")).hexdigest()
    return digest[:12]


def sessions_dir(workspace: Path) -> Path:
    return workspace / ".claw" / "sessions" / workspace_fingerprint(workspace)
```

Then:

```python
def create_session_handle(workspace: Path, session_id: str) -> SessionHandle:
    root = sessions_dir(workspace)
    root.mkdir(parents=True, exist_ok=True)

    return SessionHandle(
        id=session_id,
        path=root / f"{session_id}.jsonl",
    )
```

The important thing is that creating a handle also makes sure the folder exists.

It does not need to write the transcript yet.

The transcript is written after the runtime has something to save.

## Step 5: Bind A Session To A Workspace

A resumed coding session should not silently run inside the wrong repo.

Imagine this:

```text
session was created in:
    /work/payment-api

user accidentally resumes it from:
    /work/mobile-app
```

If the agent continues, its old file paths and repo assumptions are now wrong.

So the first line of the JSONL session should store metadata:

```json
{"kind": "metadata", "session_id": "session-abc123", "workspace_root": "/work/payment-api"}
```

When loading a session, compare:

```text
metadata workspace_root
current workspace root
```

If they do not match, stop.

That is not fancy safety logic. It is basic correctness.

## Step 6: Resolve A Resume Reference

The CLI should support three ways to resume:

```text
exact path:
    claudecode --resume .claw/sessions/a1b2c3/session-abc123.jsonl "continue"

session id:
    claudecode --resume session-abc123 "continue"

latest alias:
    claudecode --resume latest "continue"
```

Create one resolver:

```python
def resolve_session_reference(workspace: Path, reference: str) -> SessionHandle:
    candidate = Path(reference)

    if candidate.exists():
        return handle_from_path(candidate)

    if reference == "latest":
        return latest_session(workspace)

    path = sessions_dir(workspace) / f"{reference}.jsonl"
    if path.exists():
        return SessionHandle(id=reference, path=path)

    raise FileNotFoundError(f"session not found: {reference}")
```

There are three branches here.

The first branch means:

```text
the user gave us a real file path
```

The second branch means:

```text
the user wants the newest saved session in this repo
```

The final branch means:

```text
the user gave us a session id, so look for that saved id in this repo's session folder
```

Do not silently create a new session when the user asked to resume.

Do not ask the model to guess which session to resume.

This is deterministic CLI behavior.

## Step 7: Implement `latest`

The `latest` alias should choose the newest session file.

```python
def latest_session(workspace: Path) -> SessionHandle:
    files = sorted(
        sessions_dir(workspace).glob("*.jsonl"),
        key=lambda path: path.stat().st_mtime,
        reverse=True,
    )

    if not files:
        raise FileNotFoundError("no managed sessions found")

    return handle_from_path(files[0])
```

This is intentionally simple.

It looks at modification time because the newest file is usually the session you
just used.

## Step 8: Load Or Start The Transcript

Now create:

```text
claudecode/cli.py
```

The CLI needs a function like this:

```python
def load_or_create_session(workspace: Path, resume: str | None) -> Session:
    if resume is None:
        session_id = new_session_id()
        handle = create_session_handle(workspace, session_id)
        return Session.new(
            session_id=handle.id,
            workspace_root=workspace,
            persistence_path=handle.path,
        )

    handle = resolve_session_reference(workspace, resume)
    session = Session.load_jsonl(handle.path)
    ensure_workspace_matches(session, workspace)
    return session
```

Read it slowly:

```text
no --resume:
    create a fresh session id
    create a managed file path
    return an empty session bound to this workspace

with --resume:
    resolve the reference
    load that transcript
    reject it if it belongs to another workspace
```

This is the piece that makes the agent feel continuous across terminal runs.

## Step 9: Print Only The Final Assistant Text

The runtime may store several assistant messages during one turn.

For example:

```text
assistant: I will inspect the file.
assistant tool_use: read_file
tool_result: file contents
assistant: The issue is in parser.py.
```

The terminal should usually print the final assistant text, not every internal
block.

Add:

```python
def final_assistant_text(summary: TurnSummary) -> str:
    if not summary.assistant_messages:
        return ""

    final_message = summary.assistant_messages[-1]
    parts: list[str] = []

    for block in final_message.blocks:
        if isinstance(block, TextBlock):
            parts.append(block.text)

    return "".join(parts)
```

This is small, but it matters.

The session still records tool calls and tool results.

The CLI output stays readable.

## Step 10: Wire The CLI Entry Point

Use `argparse`:

```python
import argparse
import sys
from pathlib import Path


def parse_args(argv: list[str]) -> argparse.Namespace:
    parser = argparse.ArgumentParser(prog="claudecode")
    parser.add_argument("--resume")
    parser.add_argument("prompt", nargs="+")
    return parser.parse_args(argv)
```

Then:

```python
def main(argv: list[str] | None = None) -> int:
    args = parse_args(sys.argv[1:] if argv is None else argv)
    workspace = Path.cwd().resolve()
    prompt = " ".join(args.prompt)

    session = load_or_create_session(workspace, args.resume)
    runtime = build_runtime(session=session, workspace=workspace)

    summary = runtime.run_turn(prompt)
    session.save_jsonl()

    print(final_assistant_text(summary))
    return 0
```

The order matters:

```text
1. choose workspace
2. load/create session
3. build runtime with that session
4. run one turn
5. save session
6. print final answer
```

Saving after the turn means the transcript includes the user's new prompt, tool
uses, tool results, and final assistant answer.

## Step 11: Add Tests

Test new sessions:

```python
def test_new_cli_session_creates_jsonl_handle(tmp_path):
    handle = create_session_handle(tmp_path, "session-alpha")

    assert handle.id == "session-alpha"
    assert handle.path == sessions_dir(tmp_path) / "session-alpha.jsonl"
    assert handle.path.parent.exists()
```

Test `latest`:

```python
import os


def test_latest_resolves_newest_session(tmp_path):
    older = create_session_handle(tmp_path, "session-older")
    older.path.write_text('{"kind":"metadata"}\n')

    newer = create_session_handle(tmp_path, "session-newer")
    newer.path.write_text('{"kind":"metadata"}\n')
    os.utime(newer.path, (2_000_000, 2_000_000))

    resolved = latest_session(tmp_path)

    assert resolved.id == "session-newer"
```

Test workspace mismatch:

```python
def test_resume_rejects_workspace_mismatch(tmp_path):
    workspace_a = tmp_path / "repo-a"
    workspace_b = tmp_path / "repo-b"
    workspace_a.mkdir()
    workspace_b.mkdir()

    handle = create_session_handle(workspace_a, "session-alpha")
    write_metadata(handle.path, session_id="session-alpha", workspace_root=workspace_a)

    session = Session.load_jsonl(handle.path)

    with pytest.raises(ValueError, match="workspace mismatch"):
        ensure_workspace_matches(session, workspace_b)
```

Test final output:

```python
def test_final_assistant_text_uses_last_assistant_message():
    summary = TurnSummary(
        assistant_messages=[
            assistant_text("I will inspect the file."),
            assistant_text("The bug is in parser.py."),
        ],
        tool_results=[],
        iterations=1,
    )

    assert final_assistant_text(summary) == "The bug is in parser.py."
```

These tests prove the CLI behavior without needing a real model call.

## Step 12: What You Built

You added the shell around the agent.

The important pieces are:

```text
SessionHandle:
    small id/path object for a saved conversation

managed session folder:
    .claw/sessions/<workspace-fingerprint>/*.jsonl

resume resolver:
    path, session id, or latest

workspace check:
    prevents using a repo session in the wrong repo

CLI entry point:
    load session -> run runtime -> save session -> print final text
```

The model still chooses tools.

The runtime still executes the loop.

The CLI decides which conversation exists before the loop starts.

That separation is what lets a coding agent feel like one continuous teammate
instead of a fresh chatbot every time you run a command.
