# Week 16: Add Slash Commands

This week you add local commands like:

```text
/status
/diff
/session list
/resume latest
```

The new concept is command routing.

A slash command is not a normal user prompt.

This:

```text
explain this repository
```

should go to the model.

This:

```text
/status
```

should be handled by your app directly.

The model does not need to decide what `/status` means. Your CLI can parse it,
run local code, and print a report.

## Step 1: Separate Prompts From Commands

Create:

```text
claudecode/slash_commands.py
```

The first rule is simple:

```python
def looks_like_slash_command(text: str) -> bool:
    return text.strip().startswith("/")
```

Use it before calling the runtime:

```python
command = parse_slash_command(prompt)

if command is not None:
    result = run_slash_command(command, session, workspace)
    print(result.message)
else:
    summary = runtime.run_turn(prompt)
    print(final_assistant_text(summary))
```

This matters because slash commands are local app controls.

They should not consume model tokens.

They should not wait for the model to guess a tool.

They should not be recorded as normal user prompts unless you intentionally want
that history.

## Step 2: Define Command Metadata

Each command should have a small spec.

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class SlashCommandSpec:
    name: str
    summary: str
    argument_hint: str | None = None
    resume_supported: bool = False
```

The important field is `resume_supported`.

Some commands can run after loading a saved session:

```text
claudecode --resume latest /status
```

Some commands only make sense inside an interactive session.

For this lesson, define a small command table:

```python
COMMAND_SPECS = {
    "status": SlashCommandSpec(
        name="status",
        summary="Show current session status",
        resume_supported=True,
    ),
    "diff": SlashCommandSpec(
        name="diff",
        summary="Show git diff summary",
        resume_supported=True,
    ),
    "session": SlashCommandSpec(
        name="session",
        summary="Manage saved sessions",
        argument_hint="[list]",
        resume_supported=True,
    ),
    "resume": SlashCommandSpec(
        name="resume",
        summary="Switch to a saved session",
        argument_hint="<session-path|session-id|latest>",
        resume_supported=False,
    ),
}
```

This table gives your CLI enough information to render help and reject commands
that should not run in resume mode.

## Step 3: Represent Parsed Commands

Do not pass raw strings around forever.

Turn text into a typed command object.

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class StatusCommand:
    pass


@dataclass(frozen=True)
class DiffCommand:
    pass


@dataclass(frozen=True)
class SessionCommand:
    action: str | None = None


@dataclass(frozen=True)
class ResumeCommand:
    target: str
```

Then define one type alias:

```python
SlashCommand = StatusCommand | DiffCommand | SessionCommand | ResumeCommand
```

The point is not clever typing.

The point is that the rest of the app can say:

```text
if this is StatusCommand:
    render status

if this is DiffCommand:
    render diff
```

instead of repeatedly splitting strings.

## Step 4: Parse The Command Name

Start with a parser that returns `None` for normal text.

```python
def parse_slash_command(text: str) -> SlashCommand | None:
    trimmed = text.strip()

    if not trimmed.startswith("/"):
        return None

    parts = trimmed.removeprefix("/").split()
    if not parts:
        raise ValueError("slash command name is missing")

    name = parts[0]
    args = parts[1:]
```

At this point:

```text
"/status"          -> name="status", args=[]
"/session list"    -> name="session", args=["list"]
"/resume latest"   -> name="resume", args=["latest"]
```

Parsing does not run the command.

It only understands the user's text.

## Step 5: Validate Command Shapes

Now finish the parser.

```python
def parse_slash_command(text: str) -> SlashCommand | None:
    trimmed = text.strip()

    if not trimmed.startswith("/"):
        return None

    parts = trimmed.removeprefix("/").split()
    if not parts:
        raise ValueError("slash command name is missing")

    name = parts[0]
    args = parts[1:]

    if name == "status":
        require_no_args(name, args)
        return StatusCommand()

    if name == "diff":
        require_no_args(name, args)
        return DiffCommand()

    if name == "session":
        action = optional_single_arg(name, args, "[list]")
        return SessionCommand(action=action)

    if name == "resume":
        target = require_one_arg(name, args, "<session-path|session-id|latest>")
        return ResumeCommand(target=target)

    raise ValueError(f"unknown slash command: /{name}")
```

Add tiny validators:

```python
def require_no_args(name: str, args: list[str]) -> None:
    if args:
        raise ValueError(f"unexpected arguments for /{name}")


def require_one_arg(name: str, args: list[str], usage: str) -> str:
    if len(args) != 1:
        raise ValueError(f"usage: /{name} {usage}")
    return args[0]


def optional_single_arg(name: str, args: list[str], usage: str) -> str | None:
    if len(args) > 1:
        raise ValueError(f"usage: /{name} {usage}")
    return args[0] if args else None
```

This is why slash commands feel reliable.

Bad shapes fail before any work happens:

```text
/status now
```

should not be sent to the model.

It should return:

```text
unexpected arguments for /status
```

## Step 6: Define Command Results

Every slash command should return the same shape.

```python
@dataclass(frozen=True)
class SlashCommandResult:
    message: str
    session: Session
```

The `message` is what the CLI prints.

The `session` is returned because some commands change it.

For example:

```text
/resume latest
```

loads a different session, so the caller needs the new session object.

## Step 7: Render `/status`

`/status` should be local and boring in the best way.

It tells you what session is active.

```python
def render_status(session: Session, workspace: Path) -> str:
    return "\n".join(
        [
            "Status",
            f"  Workspace        {workspace}",
            f"  Session          {session.session_id}",
            f"  Messages         {len(session.messages)}",
            f"  Session file     {session.persistence_path}",
        ]
    )
```

No model call.

No tool call.

Just app state.

## Step 8: Render `/diff`

`/diff` should show the repo changes.

Use git directly:

```python
import subprocess


def render_diff(workspace: Path) -> str:
    result = subprocess.run(
        ["git", "diff", "--stat"],
        cwd=workspace,
        text=True,
        stdout=subprocess.PIPE,
        stderr=subprocess.PIPE,
        check=False,
    )

    if result.returncode != 0:
        return f"Diff\n  Error           {result.stderr.strip()}"

    output = result.stdout.strip()
    if not output:
        return "Diff\n  Result          no unstaged changes"

    return "Diff\n" + indent(output)
```

Add:

```python
def indent(text: str) -> str:
    return "\n".join(f"  {line}" for line in text.splitlines())
```

This command is useful because a coding agent constantly needs to show:

```text
what changed locally?
```

Again, the model does not need to answer that.

Git already knows.

## Step 9: Render `/session list`

From Week 15, you have a managed session folder.

Now list it:

```python
def render_session_list(workspace: Path, active_session_id: str) -> str:
    sessions = list_sessions(workspace)

    if not sessions:
        return "Sessions\n  No saved sessions"

    lines = ["Sessions"]
    for item in sessions:
        marker = "*" if item.id == active_session_id else " "
        lines.append(f"  {marker} {item.id}  messages={item.message_count}")

    return "\n".join(lines)
```

The active session marker is useful:

```text
Sessions
  * session-newer  messages=18
    session-older  messages=7
```

This is a local index of saved conversations.

It should not ask the model what sessions exist.

## Step 10: Run The Parsed Command

Now build the dispatcher.

```python
def run_slash_command(
    command: SlashCommand,
    session: Session,
    workspace: Path,
) -> SlashCommandResult:
    if isinstance(command, StatusCommand):
        return SlashCommandResult(
            message=render_status(session, workspace),
            session=session,
        )

    if isinstance(command, DiffCommand):
        return SlashCommandResult(
            message=render_diff(workspace),
            session=session,
        )

    if isinstance(command, SessionCommand):
        if command.action in (None, "list"):
            return SlashCommandResult(
                message=render_session_list(workspace, session.session_id),
                session=session,
            )
        raise ValueError("unknown /session action")

    if isinstance(command, ResumeCommand):
        resumed = load_or_create_session(workspace, resume=command.target)
        return SlashCommandResult(
            message=f"Session resumed\n  Session          {resumed.session_id}",
            session=resumed,
        )

    raise TypeError(f"unhandled slash command: {command!r}")
```

The dispatcher is intentionally direct.

Each command maps to one local function.

This is easier to test than a giant prompt that asks the model to behave like a
CLI.

## Step 11: Support Slash Commands After `--resume`

Week 15 added:

```text
claudecode --resume latest "continue the task"
```

Now add:

```text
claudecode --resume latest /status
claudecode --resume latest /diff
```

The important rule:

```text
if the prompt after --resume is a slash command,
run the slash command against the resumed session
```

Add a guard:

```python
def command_name(command: SlashCommand) -> str:
    if isinstance(command, StatusCommand):
        return "status"
    if isinstance(command, DiffCommand):
        return "diff"
    if isinstance(command, SessionCommand):
        return "session"
    if isinstance(command, ResumeCommand):
        return "resume"
    raise TypeError(f"unknown command type: {command!r}")


def ensure_resume_supported(command: SlashCommand) -> None:
    name = command_name(command)
    spec = COMMAND_SPECS[name]

    if not spec.resume_supported:
        raise ValueError(f"/{name} is interactive-only")
```

Then in the CLI:

```python
command = parse_slash_command(prompt)

if command is not None:
    if args.resume is not None:
        ensure_resume_supported(command)

    result = run_slash_command(command, session, workspace)
    session = result.session
    session.save_jsonl()
    print(result.message)
    return 0
```

This gives you a useful pattern:

```text
load session first
then run command against that session
```

So `/status` can report the restored message count, session id, and session
file.

## Step 12: Add Tests

Test normal text:

```python
def test_parse_returns_none_for_normal_prompt():
    assert parse_slash_command("explain the repo") is None
```

Test no-argument commands:

```python
def test_parse_status():
    assert parse_slash_command("/status") == StatusCommand()
```

Test invalid arguments:

```python
def test_status_rejects_args():
    with pytest.raises(ValueError, match="unexpected arguments"):
        parse_slash_command("/status now")
```

Test session list:

```python
def test_parse_session_list():
    assert parse_slash_command("/session list") == SessionCommand(action="list")
```

Test resume target:

```python
def test_parse_resume_latest():
    assert parse_slash_command("/resume latest") == ResumeCommand(target="latest")
```

Test resume support:

```python
def test_resume_command_is_not_resume_supported():
    with pytest.raises(ValueError, match="interactive-only"):
        ensure_resume_supported(ResumeCommand(target="latest"))
```

Why is `/resume` not resume-supported?

Because these are different:

```text
claudecode --resume latest /status
```

and:

```text
claudecode --resume latest /resume other-session
```

The first loads a session and reports on it.

The second tries to load one session and then switch to another session inside
the same non-interactive command. Keep that for the REPL later.

## Step 13: What You Built

You added a local command layer.

The important pieces are:

```text
SlashCommandSpec:
    command metadata and resume support

parse_slash_command:
    turns text into a typed command

validators:
    reject wrong command shapes early

run_slash_command:
    dispatches local commands without asking the model

resume-supported guard:
    allows commands like --resume latest /status safely
```

The key lesson:

```text
normal text goes to the model
slash commands go to your app
```

That one boundary makes the agent feel much more like a real developer tool.
