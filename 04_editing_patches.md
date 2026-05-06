# Week 04: Build Safe File Editing

This week teaches the fourth habit of building a Claude Code-style coding
agent:

```text
edits should be small, explicit, and reviewable
```

Week 01 built the turn loop:

```text
prompt -> model decision -> validation -> result -> transcript
```

Week 02 added repo context:

```text
repo path -> repo map -> model context -> better model decision
```

Week 03 added plan mode:

```text
normal mode -> enter plan mode -> draft plan -> exit plan mode
```

Week 04 adds controlled file editing:

```text
read file -> propose replacement -> validate patch -> apply patch -> show diff
```

The important lesson is that the model can request an edit, but the runtime owns
the filesystem. The runtime decides what paths are allowed, whether the old text
matches, how the file changes, and what diff gets shown.

Work through this file from top to bottom. Every task appears inside the lesson
at the moment you need it.

## Step 1: Understand The Job Of Week 04

A weak coding agent treats editing like this:

```text
model says "change app.py"
runtime overwrites app.py
```

That is dangerous.

A safer coding agent treats editing like this:

```text
1. read the file
2. require an exact old-text match
3. replace only that matched text
4. show the diff
5. record the edit in the transcript
```

This week, we build that safer shape.

The Week 04 rule is:

```text
never edit a file unless the runtime can explain exactly what changed
```

## Step 2: Define The Edit Request

Start with a structured edit request.

```python
from dataclasses import dataclass
from pathlib import Path
```

Now define the model's requested edit:

```python
@dataclass(frozen=True)
class FileEdit:
    path: str
    old_text: str
    new_text: str
```

Read the fields as:

```text
path:
    the repo-relative file path to edit

old_text:
    the exact text expected to already exist

new_text:
    the replacement text
```

Why require `old_text`?

```text
because the runtime should not guess where the edit belongs
```

If `old_text` is not found exactly once, the runtime should stop.

## Step 3: Define The Edit Result

The runtime should return structured output.

```python
@dataclass(frozen=True)
class EditResult:
    path: str
    changed: bool
    message: str
    diff: str
```

Read the fields as:

```text
path:
    which file was targeted

changed:
    whether the file actually changed

message:
    human-readable status

diff:
    reviewable before/after output
```

This is a pattern from the earlier weeks:

```text
model returns intent
runtime returns result
```

The model asks. The runtime reports.

## Step 4: Keep Paths Inside The Repo

Before editing, resolve the path safely.

```python
def resolve_repo_path(repo_root: Path, relative_path: str) -> Path:
    root = repo_root.resolve()
    target = (root / relative_path).resolve()
```

Break it down:

```text
repo_root.resolve():
    turns the repo root into an absolute path

(root / relative_path).resolve():
    combines the repo root and requested path, then normalizes it
```

Now block paths outside the repo:

```python
    if root not in target.parents and target != root:
        raise ValueError(f"path escapes repo: {relative_path}")

    return target
```

Why this matters:

```text
../../../secret.txt should not be editable from inside the repo
```

This is the first editing safety rail.

## Step 5: Read The Current File

Editing starts by reading.

```python
def read_text_file(path: Path) -> str:
    return path.read_text(encoding="utf-8")
```

This tiny helper is boring on purpose.

It makes later code read like a workflow:

```text
resolve path
read file
validate old text
write file
show diff
```

For Week 04, assume UTF-8 text files. Binary files, huge files, and generated
files can be handled later.

## Step 6: Validate The Old Text

The edit should only apply if `old_text` appears exactly once.

```python
def count_matches(text: str, needle: str) -> int:
    if not needle:
        return 0
    return text.count(needle)
```

Why reject empty `old_text`?

```text
empty text matches everywhere
```

Now use it before editing:

```python
def validate_edit(current_text: str, edit: FileEdit) -> str | None:
    matches = count_matches(current_text, edit.old_text)

    if matches == 0:
        return "old_text was not found"

    if matches > 1:
        return "old_text matched more than once"

    return None
```

This is the second editing safety rail:

```text
the model must identify the exact text it wants to replace
```

## Step 7: Build The New File Text

Once validation passes, replacement is simple:

```python
def apply_replacement(current_text: str, edit: FileEdit) -> str:
    return current_text.replace(edit.old_text, edit.new_text, 1)
```

The `1` matters.

```text
without 1:
    every matching occurrence would be replaced

with 1:
    only the intended match changes
```

Because `validate_edit` already required exactly one match, this is now safe and
easy to reason about.

## Step 8: Create A Diff

The user should see what changed.

Use Python's standard library:

```python
import difflib
```

Now create a small unified diff:

```python
def make_diff(path: str, before: str, after: str) -> str:
    before_lines = before.splitlines(keepends=True)
    after_lines = after.splitlines(keepends=True)

    return "".join(
        difflib.unified_diff(
            before_lines,
            after_lines,
            fromfile=f"{path} before",
            tofile=f"{path} after",
        )
    )
```

Break down the important pieces:

```text
splitlines(keepends=True):
    keeps newline characters so the diff is accurate

unified_diff:
    creates the familiar patch-style output

fromfile / tofile:
    labels the before and after versions
```

This is the third editing safety rail:

```text
every write should produce reviewable evidence
```

## Step 9: Apply One File Edit

Now combine the pieces.

```python
def apply_file_edit(repo_root: str | Path, edit: FileEdit) -> EditResult:
    root = Path(repo_root)
    path = resolve_repo_path(root, edit.path)
```

First, resolve the path safely.

Then read the file:

```python
    current_text = read_text_file(path)
```

Validate the edit:

```python
    error = validate_edit(current_text, edit)
    if error:
        return EditResult(
            path=edit.path,
            changed=False,
            message=error,
            diff="",
        )
```

If validation fails, return without writing.

Now build the replacement and diff:

```python
    new_text = apply_replacement(current_text, edit)
    diff = make_diff(edit.path, current_text, new_text)
```

Finally, write and report:

```python
    path.write_text(new_text, encoding="utf-8")
    return EditResult(
        path=edit.path,
        changed=True,
        message="edit applied",
        diff=diff,
    )
```

This function is the Week 04 product:

```text
FileEdit -> validation -> file write -> EditResult
```

## Step 10: Connect Editing To Tool Calls

In Week 01, the model could request a tool call:

```python
ToolCall(
    name="edit_file",
    arguments={
        "path": "README.md",
        "old_text": "old line",
        "new_text": "new line",
    },
)
```

Week 04 teaches what the runtime should do with that request:

```python
def edit_from_tool_call(repo_root: str | Path, arguments: dict[str, str]) -> EditResult:
    edit = FileEdit(
        path=arguments["path"],
        old_text=arguments["old_text"],
        new_text=arguments["new_text"],
    )
    return apply_file_edit(repo_root, edit)
```

This is the bridge:

```text
model asks for edit_file
runtime validates arguments
runtime applies one exact replacement
runtime returns a diff
```

Later, permissions decide whether `edit_file` is allowed in the current mode.

## Step 11: Add A Tiny CLI

Give the edit tool a command-line demo:

```python
def main() -> int:
    import argparse

    parser = argparse.ArgumentParser()
    parser.add_argument("path")
    parser.add_argument("old_text")
    parser.add_argument("new_text")
    parser.add_argument("--repo", default=".")
    args = parser.parse_args()

    result = apply_file_edit(
        args.repo,
        FileEdit(
            path=args.path,
            old_text=args.old_text,
            new_text=args.new_text,
        ),
    )
    print(result.message)
    if result.diff:
        print(result.diff)
    return 0 if result.changed else 1
```

Finish the module:

```python
if __name__ == "__main__":
    raise SystemExit(main())
```

Now the learner can run a tiny replacement:

```bash
python -m claudecode.file_edit README.md "old text" "new text"
```

This is a teaching CLI. In the real agent loop, the model requests `edit_file`
and the runtime calls `apply_file_edit`.

## Step 12: The Week 04 Build

Your build this week is one small file:

```text
claudecode/file_edit.py
```

It should contain:

```text
FileEdit
EditResult
resolve_repo_path
read_text_file
count_matches
validate_edit
apply_replacement
make_diff
apply_file_edit
edit_from_tool_call
main
```

Optional tiny smoke test:

```python
from claudecode.file_edit import FileEdit, apply_file_edit


def test_apply_file_edit_changes_one_match(tmp_path):
    target = tmp_path / "README.md"
    target.write_text("hello world\n", encoding="utf-8")

    result = apply_file_edit(
        tmp_path,
        FileEdit(
            path="README.md",
            old_text="hello",
            new_text="hi",
        ),
    )

    assert result.changed is True
    assert target.read_text(encoding="utf-8") == "hi world\n"
    assert "-hello world" in result.diff
    assert "+hi world" in result.diff
```

Add a failure test too:

```python
def test_apply_file_edit_rejects_missing_old_text(tmp_path):
    target = tmp_path / "README.md"
    target.write_text("hello world\n", encoding="utf-8")

    result = apply_file_edit(
        tmp_path,
        FileEdit(
            path="README.md",
            old_text="missing",
            new_text="replacement",
        ),
    )

    assert result.changed is False
    assert "not found" in result.message
    assert target.read_text(encoding="utf-8") == "hello world\n"
```

These tests prove the two important behaviors:

```text
valid exact edits apply
invalid edits do not write
```

## Step 13: Done Checklist

You are done when:

- file paths are resolved inside the repo
- empty `old_text` is rejected
- missing `old_text` does not write
- repeated `old_text` does not write
- a valid edit changes exactly one match
- every successful edit returns a diff
- `edit_from_tool_call` converts tool arguments into a `FileEdit`
- `python -m claudecode.file_edit ...` can run a tiny demo edit
- you can explain why the runtime, not the model, owns file writes

Stop there.

## Skool Submission

Use this format:

```text
Week 04 Submission

My safe edit tool in one sentence:

Successful edit example:

Rejected edit example:

What the diff showed:

Why exact old_text matters:

One thing I want reviewed:
```
