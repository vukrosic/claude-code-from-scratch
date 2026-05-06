# Week 08: Run Acceptance Tests

Coding agents need a simple testing habit:

```text
run command
capture structured output
summarize pass/fail
feed result back into the loop
```

In the reference shape, acceptance tests are just command strings, and the
`bash` tool returns structured fields like `stdout`, `stderr`, `interrupted`,
and `returnCodeInterpretation`.

This week builds that thin layer.

## Step 1: Define An Acceptance Test

Create:

```text
claudecode/testing.py
```

Start with one command:

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class AcceptanceTest:
    command: str
    timeout: int = 120_000
```

Read it as:

```text
command:
    the exact shell command to run

timeout:
    max runtime in milliseconds
```

Examples:

```text
pytest
cargo test --workspace
npm test
python -m pytest tests/test_file_edit.py
```

Do not invent a separate test DSL. Keep the test as the real command.

## Step 2: Extract Tests From A Task Packet

A task packet can carry acceptance tests as strings.

Use a plain dict for now:

```python
def tests_from_task_packet(packet: dict) -> list[AcceptanceTest]:
    tests = packet.get("acceptance_tests", [])
```

Reject blank values:

```python
    result = []
    for index, command in enumerate(tests):
        if not command.strip():
            raise ValueError(
                f"acceptance_tests contains an empty value at index {index}"
            )
        result.append(AcceptanceTest(command=command))

    return result
```

This mirrors the useful rule:

```text
acceptance_tests is a list of non-empty command strings
```

## Step 3: Define The Test Result

The agent should not scrape terminal output every time.

Return one small result object:

```python
@dataclass(frozen=True)
class TestRun:
    command: str
    passed: bool
    status: str
    stdout_tail: str
    stderr_tail: str
    return_code_interpretation: str | None
```

The important field is `status`:

```text
passed:
    command exited cleanly

failed:
    command returned a non-zero exit code

timeout:
    runtime interrupted the command
```

## Step 4: Keep Output Small

Test output can get huge.

Keep the last lines first:

```python
def tail_lines(text: str, max_lines: int = 40) -> str:
    lines = text.splitlines()
    return "\n".join(lines[-max_lines:])
```

Why tail instead of full output?

```text
test failures are usually near the end
the next model call has limited context
the full output can be saved later if needed
```

## Step 5: Interpret Bash Output

Use the Week 06 `BashCommandOutput` shape.

```python
from claudecode.bash_tool import BashCommandInput, BashCommandOutput, execute_bash
```

Turn a command output into a test result:

```python
def interpret_test_output(
    command: str,
    output: BashCommandOutput,
) -> TestRun:
```

Timeout first:

```python
    if output.interrupted or output.return_code_interpretation == "timeout":
        status = "timeout"
        passed = False
```

Then normal pass/fail:

```python
    elif output.return_code_interpretation is None:
        status = "passed"
        passed = True
    else:
        status = "failed"
        passed = False
```

Return a compact result:

```python
    return TestRun(
        command=command,
        passed=passed,
        status=status,
        stdout_tail=tail_lines(output.stdout),
        stderr_tail=tail_lines(output.stderr),
        return_code_interpretation=output.return_code_interpretation,
    )
```

This keeps the model from guessing what exit code `7` or a timeout means.

## Step 6: Run One Acceptance Test

Now connect the test command to the bash tool.

```python
def run_acceptance_test(test: AcceptanceTest) -> TestRun:
    output = execute_bash(
        BashCommandInput(
            command=test.command,
            timeout=test.timeout,
            description=f"Run acceptance test: {test.command}",
        )
    )

    return interpret_test_output(test.command, output)
```

The command runner stays boring on purpose:

```text
AcceptanceTest -> BashCommandInput -> BashCommandOutput -> TestRun
```

## Step 7: Run A Test List

Run the commands in order.

```python
def run_acceptance_tests(
    tests: list[AcceptanceTest],
    *,
    stop_on_failure: bool = True,
) -> list[TestRun]:
    results = []
```

Append each result:

```python
    for test in tests:
        result = run_acceptance_test(test)
        results.append(result)

        if stop_on_failure and not result.passed:
            break

    return results
```

Stopping on the first failure is useful while fixing code:

```text
first failing command gives the next thing to repair
passing earlier commands do not need to be repeated in the same loop
```

## Step 8: Summarize For The Agent Loop

The next model call should see a compact test report.

```python
def format_test_report(results: list[TestRun]) -> str:
    lines = ["Acceptance test results:"]
```

Add one status line per command:

```python
    for result in results:
        lines.append(f"- {result.status}: {result.command}")
```

Include failure details only for failing tests:

```python
        if not result.passed:
            if result.return_code_interpretation:
                lines.append(
                    f"  return_code: {result.return_code_interpretation}"
                )
            if result.stderr_tail:
                lines.append("  stderr_tail:")
                lines.append(result.stderr_tail)
            if result.stdout_tail:
                lines.append("  stdout_tail:")
                lines.append(result.stdout_tail)
```

Return the report:

```python
    return "\n".join(lines)
```

This report becomes a `tool_result` in the Week 07 loop.

## Step 9: Add A Tool Wrapper

Expose one runtime function:

```python
def run_tests_tool(arguments: dict) -> str:
    tests = [
        AcceptanceTest(
            command=item["command"],
            timeout=item.get("timeout", 120_000),
        )
        for item in arguments["tests"]
    ]

    results = run_acceptance_tests(
        tests,
        stop_on_failure=arguments.get("stop_on_failure", True),
    )

    return format_test_report(results)
```

The model-facing tool can be:

```python
RUN_TESTS_TOOL = {
    "name": "run_tests",
    "description": "Run acceptance test commands and return a compact pass/fail report.",
    "parameters": {
        "tests": "List of objects with command and optional timeout.",
        "stop_on_failure": "Whether to stop after the first failing command.",
    },
}
```

Internally, this still uses `bash`. The wrapper only makes test results easier
for the model to consume.

## Step 10: Add Tests

Create:

```text
tests/test_testing.py
```

Test task packet extraction:

```python
from claudecode.testing import tests_from_task_packet


def test_tests_from_task_packet_rejects_blank_command():
    packet = {"acceptance_tests": ["pytest", "  "]}

    try:
        tests_from_task_packet(packet)
    except ValueError as error:
        assert "empty value at index 1" in str(error)
    else:
        raise AssertionError("expected blank acceptance test to fail")
```

Test output interpretation:

```python
from claudecode.bash_tool import BashCommandOutput
from claudecode.testing import interpret_test_output


def test_interpret_test_output_marks_nonzero_as_failed():
    output = BashCommandOutput(
        stdout="",
        stderr="failed assertion",
        interrupted=False,
        background_task_id=None,
        return_code_interpretation="exit_code:1",
        no_output_expected=False,
    )

    result = interpret_test_output("pytest", output)

    assert result.passed is False
    assert result.status == "failed"
    assert result.stderr_tail == "failed assertion"
```

Test timeout:

```python
def test_interpret_test_output_marks_timeout():
    output = BashCommandOutput(
        stdout="",
        stderr="Command exceeded timeout of 100 ms",
        interrupted=True,
        background_task_id=None,
        return_code_interpretation="timeout",
        no_output_expected=False,
    )

    result = interpret_test_output("pytest", output)

    assert result.passed is False
    assert result.status == "timeout"
```

## Done Checklist

You are done when:

- acceptance tests are stored as command strings
- blank test commands are rejected
- tests run through the existing bash command output shape
- non-zero exit codes become `failed`
- timeouts become `timeout`
- passing commands become `passed`
- reports include compact stdout/stderr tails
- the report can be returned as a Week 07 `tool_result`

## Skool Submission

Use this format:

```text
Week 08 Submission

My test loop in one sentence:

Example acceptance test command:

Example failed report:

How the report returns to the model:

One thing I want reviewed:
```
