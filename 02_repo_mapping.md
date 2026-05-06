# Week 02: Build Repo Context For The Agent

This week teaches the second habit of building a Claude Code-style coding
agent:

```text
the model chooses tools better when the runtime gives it repo context
```

Week 01 built the first turn loop:

```text
prompt -> model decision -> validation -> result -> transcript
```

Week 02 adds the missing input that makes the loop feel like a coding agent:

```text
repo path -> repo map -> model context -> better model decision
```

You will build a tiny repo mapper. It will scan a project, ignore noisy folders,
identify likely entrypoints and tests, and render a short summary that can be
shown to the model in a later turn.

Work through this file from top to bottom. Every task appears inside the lesson
at the moment you need it.

## Step 1: Understand The Job Of Week 02

In Week 01, the mock model could request tools like:

```text
read_file
run_tests
```

But the model did not know anything about the repo.

That is the problem Week 02 solves. Before the model decides what to do, the
runtime should be able to summarize:

```text
where the repo is
what top-level folders exist
which files look like entrypoints
which files look like tests
how large the repo is
```

This is not tool execution yet. It is context building.

## Step 2: Learn The Repo Context Shape

Start with the output shape first.

```python
from dataclasses import dataclass
from pathlib import Path
```

These imports do two jobs:

```text
dataclass:
    lets us create small structured records

Path:
    gives us safer filesystem paths than raw strings
```

Now define the map:

```python
@dataclass(frozen=True)
class RepoMap:
    root: Path
    top_level_dirs: tuple[str, ...]
    top_level_files: tuple[str, ...]
    entrypoint_candidates: tuple[str, ...]
    test_candidates: tuple[str, ...]
    total_files: int
```

Read each field as a sentence:

```text
root:
    the repo path being summarized

top_level_dirs:
    the main project zones, like claudecode, tests, examples

top_level_files:
    metadata files, like README.md or pyproject.toml

entrypoint_candidates:
    files that might start the app or CLI

test_candidates:
    files the agent might run or inspect for verification

total_files:
    a rough size signal for the repo
```

This is the first bridge from Week 01 to Week 02:

```text
Week 01 ToolSpec tells the model what it can do.
Week 02 RepoMap tells the model where it is.
```

## Step 3: Ignore Noise Before Scanning

A coding agent should not waste context on cache folders, dependency folders,
or git internals.

Start with a small ignore list:

```python
IGNORE_DIRS = {
    ".git",
    ".venv",
    "__pycache__",
    "node_modules",
    "dist",
    "build",
}
```

Why these names?

```text
.git:
    internal git database

.venv:
    local Python environment

__pycache__:
    generated Python cache

node_modules:
    installed JavaScript dependencies

dist and build:
    generated build outputs
```

The rule is:

```text
scan source, not noise
```

## Step 4: Write A Helper For Ignored Paths

Before walking files, write one tiny helper:

```python
def is_ignored(path: Path) -> bool:
    return any(part in IGNORE_DIRS for part in path.parts)
```

Break it down:

```text
path.parts:
    turns /repo/src/main.py into pieces like "/", "repo", "src", "main.py"

any(...):
    returns True if any piece is in IGNORE_DIRS
```

Example:

```text
/repo/.git/config:
    ignored, because ".git" appears in path.parts

/repo/claudecode/agent.py:
    not ignored
```

Small helper functions like this make the bigger scanner easier to read.

## Step 5: Walk The Repo Files

Now write the file scanner:

```python
def iter_repo_files(root: Path) -> list[Path]:
    files: list[Path] = []

    for path in root.rglob("*"):
        if is_ignored(path):
            continue
        if path.is_file():
            files.append(path)

    return files
```

Read it line by line:

```text
files = []:
    start with an empty list

root.rglob("*"):
    visit everything under the repo root

if is_ignored(path):
    skip noisy paths

if path.is_file():
    keep files, not directories

return files:
    give the rest of the mapper a clean file list
```

This function is intentionally simple. Later you can add max file limits,
binary-file detection, `.gitignore` support, or token budgeting.

## Step 6: Capture Top-Level Shape

The first glance at a repo is its top level.

```python
def top_level_shape(root: Path) -> tuple[tuple[str, ...], tuple[str, ...]]:
    dirs: list[str] = []
    files: list[str] = []
```

This function returns two groups:

```text
dirs:
    top-level directories

files:
    top-level files
```

Now fill the groups:

```python
    for child in sorted(root.iterdir()):
        if child.name in IGNORE_DIRS:
            continue
        if child.is_dir():
            dirs.append(child.name)
        elif child.is_file():
            files.append(child.name)
```

Line by line:

```text
root.iterdir():
    only looks at direct children, not the whole tree

sorted(...):
    makes output stable and easy to compare

child.name in IGNORE_DIRS:
    hides noisy top-level folders

child.is_dir() / child.is_file():
    separates project zones from metadata files
```

Return immutable tuples:

```python
    return tuple(dirs), tuple(files)
```

Tuples are useful here because the map is a snapshot. The agent should not
mutate it by accident.

## Step 7: Guess Entrypoints

An entrypoint is a file that might start the app, CLI, or package.

Start with obvious names:

```python
ENTRYPOINT_NAMES = {
    "main.py",
    "cli.py",
    "app.py",
    "__main__.py",
}
```

Then search the scanned files:

```python
def find_entrypoints(root: Path, files: list[Path]) -> tuple[str, ...]:
    candidates: list[str] = []

    for path in files:
        if path.name in ENTRYPOINT_NAMES:
            candidates.append(str(path.relative_to(root)))

    return tuple(sorted(candidates))
```

Important detail:

```text
path.relative_to(root):
    turns /repo/claudecode/agent.py into claudecode/agent.py
```

Relative paths are better for model context because they are short and portable.

Call them candidates because this is a guess:

```text
entrypoint_candidates, not guaranteed_entrypoints
```

That honesty matters. Good agents do not pretend they know more than they do.

## Step 8: Find Tests

Tests are the agent's first safety rail.

Use simple patterns:

```python
def find_tests(root: Path, files: list[Path]) -> tuple[str, ...]:
    tests: list[str] = []

    for path in files:
        relative = path.relative_to(root)
        parts = relative.parts
        if "tests" in parts or path.name.startswith("test_"):
            tests.append(str(relative))

    return tuple(sorted(tests))
```

Break down the condition:

```text
"tests" in parts:
    catches tests/test_agent.py and app/tests/test_cli.py

path.name.startswith("test_"):
    catches test_agent.py even if it is not inside a tests folder
```

A real agent will eventually run tests. This week it only learns where tests
probably are.

## Step 9: Build The Repo Map

Now combine the smaller pieces:

```python
def build_repo_map(root: str | Path) -> RepoMap:
    root_path = Path(root).resolve()
    files = iter_repo_files(root_path)
    top_dirs, top_files = top_level_shape(root_path)

    return RepoMap(
        root=root_path,
        top_level_dirs=top_dirs,
        top_level_files=top_files,
        entrypoint_candidates=find_entrypoints(root_path, files),
        test_candidates=find_tests(root_path, files),
        total_files=len(files),
    )
```

Read the flow:

```text
normalize the input path
scan useful files
capture top-level shape
derive entrypoint candidates
derive test candidates
return one structured RepoMap
```

This is the full Week 02 product.

## Step 10: Render Model Context

The model should not receive the raw Python object. It needs a short text
summary.

Start with the header:

```python
def render_repo_context(repo_map: RepoMap) -> str:
    lines = [
        "Repo context:",
        f"- root: {repo_map.root}",
        f"- total files: {repo_map.total_files}",
    ]
```

Then add a tiny helper inside the function:

```python
    def add_section(title: str, values: tuple[str, ...]) -> None:
        lines.append(f"- {title}:")
        if values:
            lines.extend(f"  - {value}" for value in values[:10])
        else:
            lines.append("  - none found")
```

Why `values[:10]`?

```text
model context should be useful, not endless
```

Now add the sections:

```python
    add_section("top-level dirs", repo_map.top_level_dirs)
    add_section("top-level files", repo_map.top_level_files)
    add_section("entrypoint candidates", repo_map.entrypoint_candidates)
    add_section("test candidates", repo_map.test_candidates)
    return "\n".join(lines)
```

This renderer is the natural continuation of Week 01. Later, the turn loop can
send this string alongside the prompt and tool specs:

```text
prompt
tool specs
repo context
transcript
```

Then the model can make a better `ModelDecision`.

## Step 11: Add A Tiny CLI

Give the mapper a command-line entrypoint:

```python
def main() -> int:
    import argparse

    parser = argparse.ArgumentParser()
    parser.add_argument("path", nargs="?", default=".")
    args = parser.parse_args()

    repo_map = build_repo_map(args.path)
    print(render_repo_context(repo_map))
    return 0
```

And finish the module:

```python
if __name__ == "__main__":
    raise SystemExit(main())
```

Now the learner can run:

```bash
python -m claudecode.repo_map .
```

This is the course pattern:

```text
build a small feature
make it runnable
make its output useful to the future agent loop
```

## Step 12: The Week 02 Build

Your build this week is one small file:

```text
claudecode/repo_map.py
```

It should contain:

```text
RepoMap
IGNORE_DIRS
is_ignored
iter_repo_files
top_level_shape
ENTRYPOINT_NAMES
find_entrypoints
find_tests
build_repo_map
render_repo_context
main
```

Optional tiny smoke test:

```python
def test_repo_map_finds_entrypoints_and_tests(tmp_path):
    (tmp_path / "claudecode").mkdir()
    (tmp_path / "tests").mkdir()
    (tmp_path / "claudecode" / "cli.py").write_text("print('hi')\n")
    (tmp_path / "tests" / "test_agent.py").write_text("def test_ok(): pass\n")

    repo_map = build_repo_map(tmp_path)

    assert "claudecode" in repo_map.top_level_dirs
    assert "claudecode/cli.py" in repo_map.entrypoint_candidates
    assert "tests/test_agent.py" in repo_map.test_candidates
```

This test builds a fake repo in a temporary folder, maps it, and checks the
important outputs. It is small on purpose.

## Step 13: Done Checklist

You are done when:

- `build_repo_map` accepts a repo path
- the mapper ignores common noisy directories
- the map includes top-level directories and files
- the map includes entrypoint candidates
- the map includes test candidates
- `render_repo_context` returns short model-ready text
- `python -m claudecode.repo_map .` prints the context
- you can explain how repo context will feed the Week 01 model decision loop

Stop there.

## Skool Submission

Use this format:

```text
Week 02 Submission

My repo context builder in one sentence:

Top-level dirs it found:

Entrypoint candidates:

Test candidates:

How this connects to Week 01:

One thing I want reviewed:
```
