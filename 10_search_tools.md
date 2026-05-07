# Week 10: Search Before Reading

A coding agent should not open random files and hope.

The better habit is:

```text
glob_search -> find candidate files
grep_search -> find matching lines
read_file -> inspect the exact file window
```

This week builds two read-only search tools:

- `glob_search`: find files by path pattern
- `grep_search`: search file contents with a regex

## Step 1: Define Glob Search

Create:

```text
claudecode/search_tools.py
```

Start with the input and output:

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class GlobSearchInput:
    pattern: str
    path: str | None = None


@dataclass(frozen=True)
class GlobSearchOutput:
    num_files: int
    filenames: list[str]
    truncated: bool
```

Read it as:

```text
pattern:
    "**/*.py" or "src/**/*.ts"

path:
    optional base directory

truncated:
    true when there were more results than returned
```

Use `glob_search` when the agent knows the file shape but not the exact file.

## Step 2: Implement Glob Search

Use Python's standard `glob`.

```python
from pathlib import Path
import glob


def glob_search(input: GlobSearchInput) -> GlobSearchOutput:
    base = Path(input.path or ".").resolve()
    pattern = input.pattern
```

Build the search pattern:

```python
    search_pattern = (
        pattern
        if Path(pattern).is_absolute()
        else str(base / pattern)
    )
```

Collect files only:

```python
    matches = [
        Path(path)
        for path in glob.glob(search_pattern, recursive=True)
        if Path(path).is_file()
    ]
```

Return a bounded list:

```python
    matches = sorted(set(matches))
    limited = matches[:100]

    return GlobSearchOutput(
        num_files=len(limited),
        filenames=[str(path) for path in limited],
        truncated=len(matches) > 100,
    )
```

The limit matters. Search tools should narrow context, not dump the entire repo
into the next model call.

## Step 3: Define Grep Search

`grep_search` searches inside files.

```python
@dataclass(frozen=True)
class GrepSearchInput:
    pattern: str
    path: str | None = None
    glob: str | None = None
    output_mode: str = "files_with_matches"
    before: int = 0
    after: int = 0
    context: int = 0
    line_numbers: bool = True
    case_insensitive: bool = False
    file_type: str | None = None
    head_limit: int = 250
    offset: int = 0
    multiline: bool = False
```

The main fields:

```text
pattern:
    regex to search for

glob:
    optional file filter like "**/*.py"

output_mode:
    "files_with_matches", "content", or "count"

head_limit / offset:
    pagination so output stays small
```

## Step 4: Define Grep Output

Return structured search results:

```python
@dataclass(frozen=True)
class GrepSearchOutput:
    mode: str
    num_files: int
    filenames: list[str]
    content: str | None = None
    num_lines: int | None = None
    num_matches: int | None = None
    applied_limit: int | None = None
    applied_offset: int | None = None
```

This lets the model know:

```text
how many files matched
which files matched
whether returned output was paginated
```

## Step 5: Collect Search Files

Search either one file or a directory tree.

```python
def collect_search_files(base: Path) -> list[Path]:
    if base.is_file():
        return [base]

    return [
        path
        for path in base.rglob("*")
        if path.is_file()
    ]
```

Keep this separate because `grep_search` should read like a workflow:

```text
choose base path
compile regex
collect files
filter files
search contents
limit output
```

## Step 6: Filter Files

Support the two useful filters from the tool input:

```python
from fnmatch import fnmatch


def matches_filters(
    path: Path,
    glob_filter: str | None,
    file_type: str | None,
) -> bool:
```

Glob filter:

```python
    if glob_filter and not fnmatch(str(path), glob_filter):
        return False
```

File extension filter:

```python
    if file_type and path.suffix.lstrip(".") != file_type:
        return False

    return True
```

Examples:

```text
glob="**/*.py"
type="rs"
```

## Step 7: Add Pagination

Search can return many hits.

```python
def apply_limit(
    items: list[str],
    limit: int,
    offset: int,
) -> tuple[list[str], int | None, int | None]:
    sliced = items[offset:]
    truncated = len(sliced) > limit
    limited = sliced[:limit]

    return (
        limited,
        limit if truncated else None,
        offset if offset > 0 else None,
    )
```

`applied_limit` and `applied_offset` tell the model that more results may exist.

## Step 8: Implement Grep Search

Start with the base path and regex:

```python
import re


def grep_search(input: GrepSearchInput) -> GrepSearchOutput:
    base = Path(input.path or ".").resolve()
    flags = re.IGNORECASE if input.case_insensitive else 0
    if input.multiline:
        flags |= re.DOTALL

    regex = re.compile(input.pattern, flags)
```

Track three things:

```python
    filenames: list[str] = []
    content_lines: list[str] = []
    total_matches = 0
```

Loop through files:

```python
    for path in collect_search_files(base):
        if not matches_filters(path, input.glob, input.file_type):
            continue

        try:
            text = path.read_text(encoding="utf-8")
        except UnicodeDecodeError:
            continue
```

Skipping unreadable files is useful. Search should keep moving through the repo.

## Step 9: Support `count` Mode

`count` mode returns how many regex matches exist.

```python
        if input.output_mode == "count":
            count = len(regex.findall(text))
            if count:
                filenames.append(str(path))
                total_matches += count
            continue
```

Use this when the agent asks:

```text
how often does this symbol appear?
```

## Step 10: Support File And Content Modes

For normal search, scan line by line:

```python
        lines = text.splitlines()
        matched_indexes = [
            index
            for index, line in enumerate(lines)
            if regex.search(line)
        ]

        if not matched_indexes:
            continue

        filenames.append(str(path))
```

Default mode only needs filenames.

Content mode returns matching lines:

```python
        if input.output_mode == "content":
            context = input.context
            before = input.before or context
            after = input.after or context

            for index in matched_indexes:
                start = max(0, index - before)
                end = min(len(lines), index + after + 1)

                for line_number in range(start, end):
                    if input.line_numbers:
                        prefix = f"{path}:{line_number + 1}:"
                    else:
                        prefix = f"{path}:"
                    content_lines.append(prefix + lines[line_number])
```

The line prefix matters because the next tool call will usually be `read_file`
with a specific path and line window.

## Step 11: Return The Result

Limit filenames first:

```python
    limited_files, applied_limit, applied_offset = apply_limit(
        filenames,
        input.head_limit,
        input.offset,
    )
```

Return content mode:

```python
    if input.output_mode == "content":
        limited_lines, line_limit, line_offset = apply_limit(
            content_lines,
            input.head_limit,
            input.offset,
        )
        return GrepSearchOutput(
            mode=input.output_mode,
            num_files=len(limited_files),
            filenames=limited_files,
            content="\n".join(limited_lines),
            num_lines=len(limited_lines),
            applied_limit=line_limit,
            applied_offset=line_offset,
        )
```

Return the other modes:

```python
    return GrepSearchOutput(
        mode=input.output_mode,
        num_files=len(limited_files),
        filenames=limited_files,
        num_matches=total_matches if input.output_mode == "count" else None,
        applied_limit=applied_limit,
        applied_offset=applied_offset,
    )
```

## Step 12: Add Tool Specs

Search tools are read-only.

```python
GLOB_SEARCH_TOOL = {
    "name": "glob_search",
    "description": "Find files by glob pattern.",
    "parameters": {
        "pattern": "Glob pattern such as **/*.py.",
        "path": "Optional base directory.",
    },
}
```

```python
GREP_SEARCH_TOOL = {
    "name": "grep_search",
    "description": "Search file contents with a regex pattern.",
    "parameters": {
        "pattern": "Regex pattern.",
        "path": "Optional file or directory.",
        "glob": "Optional file glob filter.",
        "output_mode": "files_with_matches, content, or count.",
        "head_limit": "Maximum returned items.",
        "offset": "Pagination offset.",
    },
}
```

These should be allowed in read-only mode.

## Step 13: Test The Search Flow

Create:

```text
tests/test_search_tools.py
```

Test glob:

```python
from claudecode.search_tools import (
    GlobSearchInput,
    GrepSearchInput,
    glob_search,
    grep_search,
)


def test_glob_search_finds_matching_files(tmp_path):
    src = tmp_path / "src"
    src.mkdir()
    (src / "demo.py").write_text("print('hello')\n", encoding="utf-8")
    (src / "demo.md").write_text("hello\n", encoding="utf-8")

    result = glob_search(
        GlobSearchInput(pattern="**/*.py", path=str(tmp_path))
    )

    assert result.num_files == 1
    assert result.filenames[0].endswith("demo.py")
```

Test grep content mode:

```python
def test_grep_search_returns_content_lines(tmp_path):
    src = tmp_path / "src"
    src.mkdir()
    (src / "demo.py").write_text("def hello():\n    pass\n", encoding="utf-8")

    result = grep_search(
        GrepSearchInput(
            pattern="hello",
            path=str(tmp_path),
            glob="**/*.py",
            output_mode="content",
            head_limit=10,
        )
    )

    assert result.num_files == 1
    assert "demo.py:1:def hello():" in result.content
```

Test count mode:

```python
def test_grep_search_count_mode(tmp_path):
    file = tmp_path / "demo.txt"
    file.write_text("todo\ntodo\ndone\n", encoding="utf-8")

    result = grep_search(
        GrepSearchInput(
            pattern="todo",
            path=str(tmp_path),
            output_mode="count",
        )
    )

    assert result.num_matches == 2
```

## Done Checklist

You are done when:

- `glob_search` returns file paths, not file contents
- `grep_search` supports `files_with_matches`, `content`, and `count`
- search skips unreadable text files instead of crashing
- `glob` and file-type filters narrow the search space
- results are limited with `head_limit` and `offset`
- search tools are read-only
- the next step after a content hit is usually `read_file`

## Skool Submission

Use this format:

```text
Week 10 Submission

My search flow in one sentence:

Example glob_search:

Example grep_search:

When I would use read_file after search:

One thing I want reviewed:
```
