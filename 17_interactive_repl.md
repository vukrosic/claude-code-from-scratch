# Week 17: Build The Interactive REPL

This week you make the agent feel alive in the terminal.

So far, the CLI can run one prompt:

```text
claudecode "explain the repo"
```

Now add:

```text
claudecode
```

and keep a session open until the user exits.

The new concept is the REPL.

REPL means:

```text
read
evaluate
print
loop
```

For a coding agent, that means:

```text
read a line from the terminal
decide whether it is a slash command or normal prompt
run local command or model turn
print the result
repeat
```

## Step 1: Understand What Changes In Interactive Mode

One-shot mode creates a runtime, runs one prompt, saves, and exits.

Interactive mode keeps state in memory:

```text
active session
runtime
prompt history
workspace
model/settings
```

That matters because the user can do this:

```text
> explain this repo
> /status
> now add tests for the parser
> /diff
> /exit
```

Those are all part of one working session.

The agent should not create a new session for every line.

## Step 2: Create A Live CLI Object

Create:

```text
claudecode/repl.py
```

Start with a class that holds the live state.

```python
from dataclasses import dataclass, field
from pathlib import Path


@dataclass
class LiveCli:
    workspace: Path
    session: Session
    runtime: AgentRuntime
    prompt_history: list[str] = field(default_factory=list)
```

This object exists because the REPL needs mutable state.

For example:

```text
/resume latest
```

changes the active session.

And:

```text
normal user prompt
```

changes the transcript inside the runtime.

Keeping that state in one object makes the loop simple.

## Step 3: Build `LiveCli.start`

The REPL should usually start with a fresh managed session.

It should also be able to resume an old one:

```text
claudecode --resume latest
```

```python
@classmethod
def start(cls, workspace: Path, resume: str | None = None) -> "LiveCli":
    if resume is None:
        session_id = new_session_id()
        handle = create_session_handle(workspace, session_id)

        session = Session.new(
            session_id=handle.id,
            workspace_root=workspace,
            persistence_path=handle.path,
        )
    else:
        session = load_or_create_session(workspace, resume=resume)

    runtime = build_runtime(session=session, workspace=workspace)
    session.save_jsonl()

    return cls(
        workspace=workspace,
        session=session,
        runtime=runtime,
    )
```

Notice the early save.

For a fresh session, it creates the file before the first prompt.

For a resumed session, it proves the session can still be written from this
workspace.

## Step 4: Print A Startup Banner

A REPL should tell the user what world they are in.

Add:

```python
def startup_banner(self) -> str:
    return "\n".join(
        [
            "Claude Code From Scratch",
            f"  Workspace        {self.workspace}",
            f"  Session          {self.session.session_id}",
            f"  Session file     {self.session.persistence_path}",
            "",
            "Type /status for state, /diff for changes, /exit to quit.",
        ]
    )
```

This is not decoration.

It answers the basic questions:

```text
which repo am I in?
which session am I using?
how do I leave?
```

Good terminal tools make those answers obvious.

## Step 5: Run A Normal Prompt

Add a method for model turns:

```python
def run_turn(self, prompt: str) -> None:
    self.record_prompt_history(prompt)

    summary = self.runtime.run_turn(prompt)
    self.session = self.runtime.session
    self.session.save_jsonl()

    text = final_assistant_text(summary)
    if text:
        print(text)
```

Read the order carefully:

```text
1. remember the prompt in prompt history
2. run the runtime loop
3. copy the updated session out of the runtime
4. save the session
5. print the final assistant text
```

The session must be saved after every successful prompt.

If the terminal closes later, the work up to the last completed turn is still on
disk.

## Step 6: Record Prompt History

Prompt history is separate from the transcript.

The transcript stores everything:

```text
user messages
assistant messages
tool calls
tool results
```

Prompt history stores just what the user typed.

```python
def record_prompt_history(self, prompt: str) -> None:
    self.prompt_history.append(prompt)
    self.session.push_prompt_entry(prompt)
```

Why store both?

Because prompt history is for terminal ergonomics:

```text
show the last 10 prompts
search old prompts later
reuse a previous prompt
```

It should not require digging through tool calls and assistant messages.

## Step 7: Handle REPL Slash Commands

Week 16 already built `run_slash_command`.

Now wrap it for the live CLI:

```python
def handle_repl_command(self, command: SlashCommand) -> bool:
    previous_session_id = self.session.session_id

    result = run_slash_command(
        command=command,
        session=self.session,
        workspace=self.workspace,
    )

    self.session = result.session
    self.runtime = build_runtime(session=self.session, workspace=self.workspace)

    if result.message:
        print(result.message)

    return self.session.session_id != previous_session_id
```

The return value answers:

```text
did this command switch to a different session?
```

This lets `/resume latest` switch sessions and tell the loop to save state.

Commands like `/status` print a report but keep the same session.

## Step 8: Add Exit Commands

The REPL needs a predictable way out.

```python
def is_exit_command(text: str) -> bool:
    return text.strip() in {"/exit", "/quit"}
```

Handle this before normal slash command parsing.

Why before parsing?

Because `/exit` is not a model turn, and you may not want to treat it like a
normal slash command that returns a report.

It is a control signal for the loop.

## Step 9: Write The REPL Loop

Now build the loop.

```python
def run_repl(workspace: Path, resume: str | None = None) -> int:
    cli = LiveCli.start(workspace, resume=resume)
    print(cli.startup_banner())

    while True:
        try:
            line = input("> ")
        except EOFError:
            cli.session.save_jsonl()
            return 0

        text = line.strip()
        if not text:
            continue

        if is_exit_command(text):
            cli.session.save_jsonl()
            return 0

        command = parse_slash_command(text)
        if command is not None:
            changed_session = cli.handle_repl_command(command)
            if changed_session:
                cli.session.save_jsonl()
            continue

        cli.run_turn(text)
```

This is the whole REPL shape.

Read it as a decision tree:

```text
empty input:
    ignore

/exit or /quit:
    save and stop

slash command:
    handle locally

anything else:
    send to the model runtime
```

The model only sees the last branch.

That boundary is the same one you built in Week 16.

## Step 10: Connect It To The CLI

In `claudecode/cli.py`, choose REPL mode when no prompt is provided.

Change the parser so prompt is optional:

```python
parser.add_argument("prompt", nargs="*")
```

Then in `main`:

```python
def main(argv: list[str] | None = None) -> int:
    args = parse_args(sys.argv[1:] if argv is None else argv)
    workspace = Path.cwd().resolve()

    if not args.prompt:
        return run_repl(workspace, resume=args.resume)

    prompt = " ".join(args.prompt)
    session = load_or_create_session(workspace, args.resume)

    command = parse_slash_command(prompt)
    if command is not None:
        if args.resume is not None:
            ensure_resume_supported(command)
        result = run_slash_command(command, session, workspace)
        print(result.message)
        result.session.save_jsonl()
        return 0

    runtime = build_runtime(session=session, workspace=workspace)
    summary = runtime.run_turn(prompt)
    runtime.session.save_jsonl()
    print(final_assistant_text(summary))
    return 0
```

Now the CLI has two modes:

```text
claudecode
    interactive REPL

claudecode --resume latest
    interactive REPL using the newest saved session

claudecode "prompt"
    one-shot prompt
```

Both modes use the same session, runtime, and slash command pieces.

## Step 11: Keep Terminal Input Simple For Now

Real coding agents often have:

```text
arrow-key prompt history
tab completions
multi-line input
Ctrl-C behavior
```

Do not build those yet.

For this lesson, plain `input("> ")` is enough.

The important architecture is:

```text
loop
route input
save session
keep state alive
```

Fancy terminal editing can be added later without changing that core loop.

## Step 12: Add Tests

Test exit detection:

```python
def test_exit_commands():
    assert is_exit_command("/exit")
    assert is_exit_command(" /quit ")
    assert not is_exit_command("/status")
```

Test prompt history:

```python
def test_record_prompt_history(fake_live_cli):
    fake_live_cli.record_prompt_history("explain the repo")

    assert fake_live_cli.prompt_history == ["explain the repo"]
    assert fake_live_cli.session.prompt_history[-1].text == "explain the repo"
```

Test slash command routing:

```python
def test_repl_status_does_not_call_model(fake_live_cli, fake_runtime):
    command = parse_slash_command("/status")

    fake_live_cli.handle_repl_command(command)

    assert fake_runtime.run_turn_calls == []
```

Test normal prompt routing:

```python
def test_repl_prompt_calls_runtime(fake_live_cli, fake_runtime):
    fake_live_cli.run_turn("explain the repo")

    assert fake_runtime.run_turn_calls == ["explain the repo"]
```

Test the loop with fake input:

```python
def test_repl_exits_on_exit(monkeypatch, tmp_path, fake_live_cli):
    inputs = iter(["/exit"])
    monkeypatch.setattr("builtins.input", lambda _: next(inputs))
    monkeypatch.setattr(
        LiveCli,
        "start",
        classmethod(lambda cls, workspace, resume=None: fake_live_cli),
    )

    assert run_repl(tmp_path, resume=None) == 0
```

The test replaces `LiveCli.start` so it does not create a real runtime.

These tests prove the routing without requiring a real model.

## Step 13: What You Built

You built the interactive shell around the agent.

The important pieces are:

```text
LiveCli:
    keeps session, runtime, workspace, and prompt history alive

startup banner:
    tells the user what session and repo they are in

run_turn:
    records prompt history, runs the model loop, saves session

handle_repl_command:
    runs local slash commands without calling the model

run_repl:
    reads terminal input until /exit, /quit, or EOF
```

The key lesson:

```text
one-shot CLI runs one turn
interactive REPL keeps the agent alive across many turns
```

That is a major shift.

The agent is no longer just a command.

It is now a persistent terminal workspace.
