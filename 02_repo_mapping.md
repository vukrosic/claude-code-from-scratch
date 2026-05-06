# Week 02: Build A Repo Mapper Before Editing

This week teaches the second habit of building a Claude Code-style coding
agent:

```text
the agent should map the repo before it plans a change
```

You will build a tiny repo mapper. It will count files, identify important
directories, guess likely entrypoints, and produce a short markdown summary.
No file editing yet.

Work through this file from top to bottom. Every task appears inside the lesson
at the moment you need it.

## Step 1: Understand The Job Of Week 02

Week 01 built the turn loop:

```text
prompt -> route -> result -> transcript
```

Week 02 adds repo context:

```text
repo path -> file scan -> repo map -> agent turn
```

A coding agent should not start with:

```text
I will edit some file that sounds relevant.
```

It should start with:

```text
I know the repo roots, likely entrypoints, test locations, and current file map.
```

That is what this lesson builds.

## Step 2: Learn The Repo Map Shape

A useful repo map is short. It should answer:

```text
where is the repo?
what are the top-level directories?
where does the app likely start?
where are the tests?
what files look important?
what should the agent inspect before editing?
```

Use a data model instead of loose strings:

```python
from dataclasses import dataclass
from pathlib import Path


@dataclass(frozen=True)
class RepoMap:
    root: Path
    top_level_dirs: tuple[str, ...]
    top_level_files: tuple[str, ...]
    entrypoint_candidates: tuple[str, ...]
    test_candidates: tuple[str, ...]
    total_files: int
```

This gives the agent a structured object it can pass into later planning code.

## Step 3: Walk The Repo Safely

Start with a small scanner:

```python
IGNORE_DIRS = {
    ".git",
    ".venv",
    "__pycache__",
    "node_modules",
    "dist",
    "build",
}


def iter_repo_files(root: Path) -> list[Path]:
    files: list[Path] = []
    for path in root.rglob("*"):
        if any(part in IGNORE_DIRS for part in path.parts):
            continue
        if path.is_file():
            files.append(path)
    return files
```

The ignore list matters. A coding agent should avoid wasting context on build
artifacts, dependency folders, caches, and git internals.

This is the first repo-mapping rule:

```text
scan source, not noise
```

## Step 4: Identify Top-Level Shape

Now collect top-level files and directories:

```python
def top_level_shape(root: Path) -> tuple[tuple[str, ...], tuple[str, ...]]:
    dirs: list[str] = []
    files: list[str] = []

    for child in sorted(root.iterdir()):
        if child.name in IGNORE_DIRS:
            continue
        if child.is_dir():
            dirs.append(child.name)
        elif child.is_file():
            files.append(child.name)

    return tuple(dirs), tuple(files)
```

Read this like a first glance at the project:

```text
directories tell you the project zones
files tell you the project metadata
```

For example:

```text
claudecode/
examples/
tests/
README.md
pyproject.toml
```

That already tells you this is probably a Python project with package code,
examples, tests, and install metadata.

## Step 5: Guess Entrypoints

An entrypoint is where execution may begin.

Start with obvious filename patterns:

```python
ENTRYPOINT_NAMES = {
    "main.py",
    "cli.py",
    "app.py",
    "__main__.py",
}


def find_entrypoints(root: Path, files: list[Path]) -> tuple[str, ...]:
    candidates: list[str] = []
    for path in files:
        if path.name in ENTRYPOINT_NAMES:
            candidates.append(str(path.relative_to(root)))
    return tuple(sorted(candidates))
```

This is only a guess. Good repo mapping is honest about guesses.

The right wording is:

```text
entrypoint candidates
```

not:

```text
the entrypoint
```

That small distinction keeps the agent from pretending certainty too early.

## Step 6: Find Tests

Tests are the agent's first safety rail.

Use common patterns:

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

A coding agent should always ask:

```text
how will I know if my change worked?
```

Finding tests does not prove the repo is safe to edit. It gives the agent a
starting point for verification.

## Step 7: Build The Repo Map

Now combine the pieces:

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

This is the first useful repo context object for your agent.

Later, the turn loop can do this:

```text
receive prompt
build repo map
route prompt with repo context
choose first files to inspect
```

Week 02 stops at the repo map.

## Step 8: Render The Map For Humans

Agents need structured data. Humans need readable summaries.

Add a markdown renderer:

```python
def render_repo_map(repo_map: RepoMap) -> str:
    lines = [
        "# Repo Map",
        "",
        f"Root: `{repo_map.root}`",
        f"Total files: {repo_map.total_files}",
        "",
        "## Top-Level Directories",
        *(f"- `{name}`" for name in repo_map.top_level_dirs),
        "",
        "## Top-Level Files",
        *(f"- `{name}`" for name in repo_map.top_level_files),
        "",
        "## Entrypoint Candidates",
        *(f"- `{path}`" for path in repo_map.entrypoint_candidates),
        "",
        "## Test Candidates",
        *(f"- `{path}`" for path in repo_map.test_candidates),
    ]
    return "\n".join(lines)
```

This gives the agent something it can show in a terminal, write to a note, or
include in a planning step.

## Step 9: Add A Tiny CLI

Give the mapper a command-line entrypoint:

```python
def main() -> int:
    import argparse

    parser = argparse.ArgumentParser()
    parser.add_argument("path", nargs="?", default=".")
    args = parser.parse_args()

    repo_map = build_repo_map(args.path)
    print(render_repo_map(repo_map))
    return 0


if __name__ == "__main__":
    raise SystemExit(main())
```

Now the learner can run:

```bash
python -m claudecode.repo_map .
```

This is the second course pattern:

```text
build a small feature
make it runnable
make its output readable
```

## Step 10: Connect Repo Mapping To The Agent Loop

Week 01 routed only from the prompt. Week 02 prepares the next upgrade:

```python
@dataclass(frozen=True)
class AgentContext:
    repo_map: RepoMap
```

In a later lesson, the agent turn will accept context:

```python
def run_turn(self, prompt: str, context: AgentContext) -> TurnResult:
    ...
```

That is the bridge from chatbot to coding agent:

```text
the agent responds with knowledge of the repo it is inside
```

Do not wire everything together yet. Keep the repo mapper independent and easy
to test.

## Step 11: The Week 02 Build

Your build this week is one small file:

```text
claudecode/repo_map.py
```

It should contain:

```text
RepoMap
IGNORE_DIRS
iter_repo_files
top_level_shape
find_entrypoints
find_tests
build_repo_map
render_repo_map
main
```

Optional tiny smoke test:

```python
def test_repo_map_finds_tests(tmp_path):
    (tmp_path / "src").mkdir()
    (tmp_path / "tests").mkdir()
    (tmp_path / "src" / "main.py").write_text("print('hi')\n")
    (tmp_path / "tests" / "test_app.py").write_text("def test_ok(): pass\n")

    repo_map = build_repo_map(tmp_path)

    assert "src" in repo_map.top_level_dirs
    assert "tests/test_app.py" in repo_map.test_candidates
    assert "src/main.py" in repo_map.entrypoint_candidates
```

## Step 12: Done Checklist

You are done when:

- `build_repo_map` accepts a repo path
- the mapper ignores common noisy directories
- the map includes top-level directories and files
- the map includes entrypoint candidates
- the map includes test candidates
- `render_repo_map` returns readable markdown
- `python -m claudecode.repo_map .` prints the map

Stop there.

## Skool Submission

Use this format:

```text
Week 02 Submission

My repo mapper in one sentence:

Top-level dirs it found:

Entrypoint candidates:

Test candidates:

One thing I want reviewed:
```
