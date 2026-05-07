# Week 11: Read File Windows

After search finds a file, the agent should not always read the whole file.

The useful pattern is:

```text
grep_search finds path:line
read_file reads a small window around that line
edit_file uses exact text from that window
```

This week builds `read_file` with `offset` and `limit`.

## Step 1: Define The Input

Create:

```text
claudecode/read_file.py
```

Start with the request:

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class ReadFileInput:
    path: str
    offset: int | None = None
    limit: int | None = None
```

Read it as:

```text
path:
    file to read

offset:
    zero-based line offset

limit:
    number of lines to return
```

The offset is zero-based because it is used for slicing.

## Step 2: Define The Output

Return the text plus line metadata.

```python
@dataclass(frozen=True)
class TextFilePayload:
    file_path: str
    content: str
    num_lines: int
    start_line: int
    total_lines: int
```

Wrap it in a result:

```python
@dataclass(frozen=True)
class ReadFileOutput:
    kind: str
    file: TextFilePayload
```

The important fields:

```text
content:
    selected file text

start_line:
    one-based line number shown to the model/user

total_lines:
    full file length
```

Input `offset` is zero-based. Output `start_line` is one-based.

## Step 3: Resolve The Path

Use the same basic path rule as earlier tools.

```python
from pathlib import Path


def resolve_path(path: str) -> Path:
    candidate = Path(path)
    if not candidate.is_absolute():
        candidate = Path.cwd() / candidate

    return candidate.resolve()
```

For this lesson, keep it simple.

In a full workspace runtime, this is also where you enforce:

```text
the resolved path must stay inside the workspace root
```

## Step 4: Reject Huge Files

Do not let `read_file` dump massive files into the model.

```python
MAX_READ_SIZE = 10 * 1024 * 1024


def validate_size(path: Path) -> None:
    size = path.stat().st_size
    if size > MAX_READ_SIZE:
        raise ValueError(
            f"file is too large ({size} bytes, max {MAX_READ_SIZE})"
        )
```

This is not just safety. It protects the next model call from being flooded.

## Step 5: Reject Binary Files

Check the first chunk for NUL bytes.

```python
def is_binary_file(path: Path) -> bool:
    with path.open("rb") as file:
        chunk = file.read(8192)

    return b"\x00" in chunk
```

Use it before reading text:

```python
def validate_text_file(path: Path) -> None:
    validate_size(path)

    if is_binary_file(path):
        raise ValueError("file appears to be binary")
```

Coding agents should read source files, not images, lockfiles with binary bytes,
or compiled artifacts.

## Step 6: Read Lines

Now read the text.

```python
def read_lines(path: Path) -> list[str]:
    content = path.read_text(encoding="utf-8")
    return content.splitlines()
```

Using `splitlines()` makes line windows straightforward:

```text
lines[0] is line 1
lines[1] is line 2
```

## Step 7: Select The Window

Convert `offset` and `limit` into slice indexes.

```python
def select_window(
    lines: list[str],
    offset: int | None,
    limit: int | None,
) -> tuple[list[str], int]:
    start_index = min(offset or 0, len(lines))
```

If `limit` is missing, read to the end:

```python
    if limit is None:
        end_index = len(lines)
    else:
        end_index = min(start_index + limit, len(lines))
```

Return selected lines and the start index:

```python
    return lines[start_index:end_index], start_index
```

Example:

```text
offset=10, limit=5
returns lines 11 through 15
```

## Step 8: Implement `read_file`

Combine the pieces.

```python
def read_file(input: ReadFileInput) -> ReadFileOutput:
    path = resolve_path(input.path)
    validate_text_file(path)

    lines = read_lines(path)
    selected, start_index = select_window(
        lines,
        input.offset,
        input.limit,
    )
```

Build the payload:

```python
    payload = TextFilePayload(
        file_path=str(path),
        content="\n".join(selected),
        num_lines=len(selected),
        start_line=start_index + 1,
        total_lines=len(lines),
    )
```

Return the result:

```python
    return ReadFileOutput(kind="text", file=payload)
```

That is the whole tool:

```text
path -> validate -> lines -> selected window -> line metadata
```

## Step 9: Convert Search Hits Into Read Windows

`grep_search` content mode returns lines like:

```text
src/agent.py:42:def run_turn(...):
```

To inspect around line 42, call:

```python
ReadFileInput(
    path="src/agent.py",
    offset=41,
    limit=40,
)
```

Why `41`?

```text
line numbers are one-based
offset is zero-based
42 - 1 = 41
```

If you want context before the hit:

```python
line_number = 42
before = 10
offset = max(0, line_number - before - 1)
limit = 30
```

That reads roughly 10 lines before and 20 lines after.

## Step 10: Add The Tool Spec

The tool is read-only.

```python
READ_FILE_TOOL = {
    "name": "read_file",
    "description": "Read a text file from the workspace.",
    "parameters": {
        "path": "File path to read.",
        "offset": "Optional zero-based line offset.",
        "limit": "Optional number of lines to return.",
    },
}
```

The model should use this after search narrows the target.

## Step 11: Test Windowed Reading

Create:

```text
tests/test_read_file.py
```

Read one line from the middle:

```python
from claudecode.read_file import ReadFileInput, read_file


def test_read_file_returns_line_window(tmp_path, monkeypatch):
    monkeypatch.chdir(tmp_path)
    path = tmp_path / "demo.txt"
    path.write_text("one\ntwo\nthree\n", encoding="utf-8")

    output = read_file(
        ReadFileInput(
            path="demo.txt",
            offset=1,
            limit=1,
        )
    )

    assert output.kind == "text"
    assert output.file.content == "two"
    assert output.file.start_line == 2
    assert output.file.num_lines == 1
    assert output.file.total_lines == 3
```

Test reading to the end:

```python
def test_read_file_without_limit_reads_to_end(tmp_path, monkeypatch):
    monkeypatch.chdir(tmp_path)
    path = tmp_path / "demo.txt"
    path.write_text("one\ntwo\nthree\n", encoding="utf-8")

    output = read_file(
        ReadFileInput(
            path="demo.txt",
            offset=1,
        )
    )

    assert output.file.content == "two\nthree"
    assert output.file.start_line == 2
```

Test binary rejection:

```python
def test_read_file_rejects_binary_file(tmp_path, monkeypatch):
    monkeypatch.chdir(tmp_path)
    path = tmp_path / "demo.bin"
    path.write_bytes(b"\x00\x01\x02")

    try:
        read_file(ReadFileInput(path="demo.bin"))
    except ValueError as error:
        assert "binary" in str(error)
    else:
        raise AssertionError("expected binary file to fail")
```

## Done Checklist

You are done when:

- `read_file` accepts `path`, `offset`, and `limit`
- `offset` is zero-based
- returned `start_line` is one-based
- large files are rejected
- binary files are rejected
- the output includes `content`, `num_lines`, `start_line`, and `total_lines`
- search hits can be turned into read windows

## Skool Submission

Use this format:

```text
Week 11 Submission

My read window rule in one sentence:

Example grep hit:

ReadFileInput for that hit:

Why offset and start_line differ:

One thing I want reviewed:
```
