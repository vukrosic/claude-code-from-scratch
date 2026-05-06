# Week 04: Safe File Editing

A coding agent becomes dangerous the moment it can write files.

So this week is about one rule:

```text
the model may request an edit, but the runtime owns the write
```

We will build two file tools:

- `edit_file`: replace text inside an existing file
- `write_file`: create or overwrite a whole file

Use `edit_file` for small targeted changes. Use `write_file` when the whole file
is new or the replacement would be massive.

## Step 1: Define The Edit Tool Input

Start with the same shape used by many coding-agent file editors:

```python
from dataclasses import dataclass
from pathlib import Path
```

```python
@dataclass(frozen=True)
class FileEdit:
    path: str
    old_string: str
    new_string: str
    replace_all: bool = False
```

Read it as:

```text
path:
    repo-relative file path

old_string:
    exact text to find

new_string:
    replacement text

replace_all:
    false means targeted edit
    true means replace every occurrence
```

The important idea:

```text
edit_file does not say "change line 42"
edit_file says "find this exact text and replace it"
```

Line numbers drift. Exact strings prove the model is editing text that actually
exists.

## Step 2: Define Whole-File Write Input

Large replacements should not be squeezed into `old_string`.

For those, use a separate full-file write request:

```python
@dataclass(frozen=True)
class FileWrite:
    path: str
    content: str
```

Use this when:

- creating a new file
- replacing almost the entire file
- generating a small complete module from scratch

The split is simple:

```text
edit_file:
    patch one known area

write_file:
    replace the whole file content
```

## Step 3: Return A Reviewable Result

Both tools should return the same result shape.

```python
@dataclass(frozen=True)
class FileResult:
    path: str
    changed: bool
    message: str
    diff: str
```

The runtime should never silently write. It should always be able to show what
changed.

## Step 4: Keep Paths Inside The Repo

Never trust a path from the model.

```python
def resolve_repo_path(repo_root: Path, relative_path: str) -> Path:
    root = repo_root.resolve()
    target = (root / relative_path).resolve()

    if target != root and root not in target.parents:
        raise ValueError(f"path escapes repo: {relative_path}")

    return target
```

This blocks paths like:

```text
../../private-notes.txt
```

The model can ask. The runtime can refuse.

## Step 5: Read Text Files Only

Keep the first version narrow.

```python
def read_text_file(path: Path) -> str:
    return path.read_text(encoding="utf-8")
```

For now, assume normal UTF-8 source files.

Later you can add binary detection, file-size limits, generated-file deny lists,
and permission prompts. Do not add those yet.

## Step 6: Validate An Edit

First reject edits that cannot be meaningful.

```python
def validate_edit(current_text: str, edit: FileEdit) -> str | None:
    if not edit.old_string:
        return "old_string must not be empty"

    if edit.old_string == edit.new_string:
        return "old_string and new_string must differ"
```

Now count matches:

```python
    matches = current_text.count(edit.old_string)

    if matches == 0:
        return "old_string was not found"
```

Here is the safety choice for this course:

```python
    if matches > 1 and not edit.replace_all:
        return "old_string matched more than once; use replace_all=True or provide more context"

    return None
```

Some agents replace only the first match by default. For learning, this course is
stricter: if the text appears multiple times, make the model provide a more
specific `old_string` or explicitly choose `replace_all=True`.

That makes mistakes easier to see.

## Step 7: Apply The Replacement

Now the edit behavior is tiny.

```python
def apply_replacement(current_text: str, edit: FileEdit) -> str:
    if edit.replace_all:
        return current_text.replace(edit.old_string, edit.new_string)

    return current_text.replace(edit.old_string, edit.new_string, 1)
```

Read it as:

```text
replace_all=True:
    change every occurrence

replace_all=False:
    change one occurrence
```

Because validation already rejected repeated matches in single-edit mode, the
final `replace(..., 1)` changes exactly one place.

## Step 8: Build A Diff

Use the standard library.

```python
import difflib
```

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

This gives the user reviewable evidence:

```text
- old line
+ new line
```

The runtime should generate the diff before it writes.

## Step 9: Implement `edit_file`

The edit tool gets a `write` flag so you can preview first.

```python
def edit_file(
    repo_root: str | Path,
    edit: FileEdit,
    *,
    write: bool,
) -> FileResult:
```

Resolve and read:

```python
    root = Path(repo_root)
    path = resolve_repo_path(root, edit.path)
    current_text = read_text_file(path)
```

Validate before changing anything:

```python
    error = validate_edit(current_text, edit)
    if error:
        return FileResult(edit.path, False, error, "")
```

Build the proposed file:

```python
    new_text = apply_replacement(current_text, edit)
    diff = make_diff(edit.path, current_text, new_text)
```

Preview mode stops here:

```python
    if not write:
        return FileResult(edit.path, False, "edit previewed", diff)
```

Write only after the diff exists:

```python
    path.write_text(new_text, encoding="utf-8")
    return FileResult(edit.path, True, "edit applied", diff)
```

The flow is:

```text
read -> validate -> build diff -> optionally write
```

## Step 10: Implement `write_file`

Whole-file writes use the same safety shape.

```python
def write_file(
    repo_root: str | Path,
    file_write: FileWrite,
    *,
    write: bool,
) -> FileResult:
```

Resolve the path and read the old content if the file exists:

```python
    root = Path(repo_root)
    path = resolve_repo_path(root, file_write.path)
    old_text = path.read_text(encoding="utf-8") if path.exists() else ""
```

Create the diff:

```python
    diff = make_diff(file_write.path, old_text, file_write.content)
```

Preview mode:

```python
    if not write:
        return FileResult(file_write.path, False, "write previewed", diff)
```

Write the full file:

```python
    path.parent.mkdir(parents=True, exist_ok=True)
    path.write_text(file_write.content, encoding="utf-8")
    return FileResult(file_write.path, True, "file written", diff)
```

Notice the difference:

```text
edit_file needs old_string
write_file needs complete content
```

## Step 11: Connect Tool Calls To Runtime Functions

The model-facing specs can be simple.

```python
EDIT_FILE_TOOL = {
    "name": "edit_file",
    "description": "Replace exact text in an existing repo file.",
    "parameters": {
        "path": "Repo-relative path.",
        "old_string": "Exact text to replace.",
        "new_string": "Replacement text.",
        "replace_all": "Whether to replace every occurrence.",
    },
}
```

```python
WRITE_FILE_TOOL = {
    "name": "write_file",
    "description": "Create or overwrite a text file.",
    "parameters": {
        "path": "Repo-relative path.",
        "content": "Complete file content.",
    },
}
```

The runtime still owns conversion and validation.

```python
def edit_from_tool_call(
    repo_root: str | Path,
    arguments: dict,
    *,
    write: bool,
) -> FileResult:
    edit = FileEdit(
        path=arguments["path"],
        old_string=arguments["old_string"],
        new_string=arguments["new_string"],
        replace_all=arguments.get("replace_all", False),
    )
    return edit_file(repo_root, edit, write=write)
```

```python
def write_from_tool_call(
    repo_root: str | Path,
    arguments: dict,
    *,
    write: bool,
) -> FileResult:
    file_write = FileWrite(
        path=arguments["path"],
        content=arguments["content"],
    )
    return write_file(repo_root, file_write, write=write)
```

The model chooses the tool. The runtime decides whether the write is allowed.

## Step 12: Add A Tiny CLI

Use the CLI to practice the edit primitive.

```python
def main() -> int:
    import argparse

    parser = argparse.ArgumentParser()
    parser.add_argument("path")
    parser.add_argument("old_string")
    parser.add_argument("new_string")
    parser.add_argument("--repo", default=".")
    parser.add_argument("--replace-all", action="store_true")
    parser.add_argument("--write", action="store_true")
    args = parser.parse_args()

    result = edit_file(
        args.repo,
        FileEdit(
            path=args.path,
            old_string=args.old_string,
            new_string=args.new_string,
            replace_all=args.replace_all,
        ),
        write=args.write,
    )

    print(result.message)
    if result.diff:
        print(result.diff)

    return 0 if result.diff or result.changed else 1
```

Finish the file:

```python
if __name__ == "__main__":
    raise SystemExit(main())
```

Preview:

```bash
python -m claudecode.file_edit README.md "old text" "new text"
```

Apply:

```bash
python -m claudecode.file_edit README.md "old text" "new text" --write
```

Replace every match:

```bash
python -m claudecode.file_edit README.md "old text" "new text" --replace-all --write
```

## Step 13: Test The Important Behavior

Create:

```text
claudecode/file_edit.py
```

Add a successful targeted edit test:

```python
from claudecode.file_edit import FileEdit, FileWrite, edit_file, write_file


def test_edit_file_changes_one_exact_match(tmp_path):
    target = tmp_path / "README.md"
    target.write_text("hello world\n", encoding="utf-8")

    result = edit_file(
        tmp_path,
        FileEdit(
            path="README.md",
            old_string="hello",
            new_string="hi",
        ),
        write=True,
    )

    assert result.changed is True
    assert target.read_text(encoding="utf-8") == "hi world\n"
    assert "-hello world" in result.diff
    assert "+hi world" in result.diff
```

Add a repeated-match rejection test:

```python
def test_edit_file_rejects_repeated_match_without_replace_all(tmp_path):
    target = tmp_path / "README.md"
    target.write_text("alpha\nbeta\nalpha\n", encoding="utf-8")

    result = edit_file(
        tmp_path,
        FileEdit(
            path="README.md",
            old_string="alpha",
            new_string="omega",
        ),
        write=True,
    )

    assert result.changed is False
    assert "more than once" in result.message
    assert target.read_text(encoding="utf-8") == "alpha\nbeta\nalpha\n"
```

Add a `replace_all` test:

```python
def test_edit_file_replace_all_changes_every_match(tmp_path):
    target = tmp_path / "README.md"
    target.write_text("alpha\nbeta\nalpha\n", encoding="utf-8")

    result = edit_file(
        tmp_path,
        FileEdit(
            path="README.md",
            old_string="alpha",
            new_string="omega",
            replace_all=True,
        ),
        write=True,
    )

    assert result.changed is True
    assert target.read_text(encoding="utf-8") == "omega\nbeta\nomega\n"
```

Add a whole-file write test:

```python
def test_write_file_replaces_whole_file(tmp_path):
    target = tmp_path / "README.md"
    target.write_text("old file\n", encoding="utf-8")

    result = write_file(
        tmp_path,
        FileWrite(path="README.md", content="new file\n"),
        write=True,
    )

    assert result.changed is True
    assert target.read_text(encoding="utf-8") == "new file\n"
    assert "-old file" in result.diff
    assert "+new file" in result.diff
```

Stop when these behaviors are true:

- `edit_file` replaces exact text
- repeated text is rejected unless `replace_all=True`
- `replace_all=True` changes every match
- `write_file` replaces the whole file
- both tools can preview with a diff before writing
- paths cannot escape the repo

## Skool Submission

Use this format:

```text
Week 04 Submission

My safe file tools in one sentence:

Targeted edit example:

Replace-all example:

Whole-file write example:

Why edit_file and write_file are separate tools:
```
