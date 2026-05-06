# Week 06: Run Bash Commands

This week builds the `bash` tool.

The job is simple:

```text
command in -> structured command result out
```

Do not make the agent parse terminal text by guessing. Return fields the agent
can reason about.

## Step 1: Define The Input

Create:

```text
claudecode/bash_tool.py
```

Start with the command request:

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class BashCommandInput:
    command: str
    timeout: int | None = None
    description: str | None = None
    run_in_background: bool = False
```

Read it as:

```text
command:
    shell command to run

timeout:
    optional timeout in milliseconds

description:
    optional human label for the command

run_in_background:
    true means start the process and return immediately
```

## Step 2: Define The Output

Return structured output:

```python
@dataclass(frozen=True)
class BashCommandOutput:
    stdout: str
    stderr: str
    interrupted: bool
    background_task_id: str | None
    return_code_interpretation: str | None
    no_output_expected: bool
```

The useful fields are:

```text
stdout / stderr:
    captured process output

interrupted:
    true when timeout stopped the command

background_task_id:
    process id when running in background

return_code_interpretation:
    None for exit 0, "exit_code:7" for failure, "timeout" for timeout
```

## Step 3: Run A Background Command

Background commands do not wait for output.

```python
import subprocess


def run_background(command: str) -> BashCommandOutput:
    process = subprocess.Popen(
        command,
        shell=True,
        stdout=subprocess.DEVNULL,
        stderr=subprocess.DEVNULL,
    )

    return BashCommandOutput(
        stdout="",
        stderr="",
        interrupted=False,
        background_task_id=str(process.pid),
        return_code_interpretation=None,
        no_output_expected=True,
    )
```

This is useful for long-running servers later.

Example:

```text
python -m http.server 8000
```

The agent should get a task id immediately instead of hanging forever.

## Step 4: Run A Foreground Command

Foreground commands wait and capture output.

```python
def run_foreground(command: str, timeout: int | None) -> BashCommandOutput:
    timeout_seconds = None if timeout is None else timeout / 1000
```

Run the command:

```python
    try:
        completed = subprocess.run(
            command,
            shell=True,
            text=True,
            capture_output=True,
            timeout=timeout_seconds,
        )
```

Interpret the exit code:

```python
        return_code = completed.returncode
        interpretation = None if return_code == 0 else f"exit_code:{return_code}"

        return BashCommandOutput(
            stdout=completed.stdout,
            stderr=completed.stderr,
            interrupted=False,
            background_task_id=None,
            return_code_interpretation=interpretation,
            no_output_expected=not completed.stdout.strip()
            and not completed.stderr.strip(),
        )
```

Failed commands still return structured output. They are not Python exceptions.

## Step 5: Return Timeout As Output

Timeout is also a structured result.

```python
    except subprocess.TimeoutExpired:
        return BashCommandOutput(
            stdout="",
            stderr=f"Command exceeded timeout of {timeout} ms",
            interrupted=True,
            background_task_id=None,
            return_code_interpretation="timeout",
            no_output_expected=True,
        )
```

This lets the agent know:

```text
the command did not fail normally
it was interrupted by the runtime
```

## Step 6: Build `execute_bash`

Now choose background or foreground:

```python
def execute_bash(input: BashCommandInput) -> BashCommandOutput:
    if input.run_in_background:
        return run_background(input.command)

    return run_foreground(input.command, input.timeout)
```

That is the whole tool.

Later, the permission gate from Week 05 should run before this function.

## Step 7: Add Tests

Create:

```text
tests/test_bash_tool.py
```

Successful command:

```python
from claudecode.bash_tool import BashCommandInput, execute_bash


def test_bash_success_captures_stdout():
    output = execute_bash(
        BashCommandInput(command="printf 'hello'")
    )

    assert output.stdout == "hello"
    assert output.stderr == ""
    assert output.interrupted is False
    assert output.return_code_interpretation is None
```

Failed command:

```python
def test_bash_failure_returns_exit_code_and_stderr():
    output = execute_bash(
        BashCommandInput(command="printf 'oops' >&2; exit 7")
    )

    assert "oops" in output.stderr
    assert output.return_code_interpretation == "exit_code:7"
```

Timeout:

```python
def test_bash_timeout_returns_interrupted_output():
    output = execute_bash(
        BashCommandInput(command="sleep 1", timeout=10)
    )

    assert output.interrupted is True
    assert output.return_code_interpretation == "timeout"
    assert "Command exceeded timeout" in output.stderr
```

Background:

```python
def test_bash_background_returns_process_id():
    output = execute_bash(
        BashCommandInput(command="sleep 1", run_in_background=True)
    )

    assert output.background_task_id is not None
    assert output.no_output_expected is True
```

Stop when these are true:

- successful commands capture `stdout`
- failing commands return `exit_code:N`
- timeouts return `interrupted=True`
- background commands return a process id
- command results are structured, not free-form terminal text

## Skool Submission

Use this format:

```text
Week 06 Submission

My bash tool in one sentence:

Success output example:

Failure output example:

Timeout output example:

Why bash returns structured output:
```
