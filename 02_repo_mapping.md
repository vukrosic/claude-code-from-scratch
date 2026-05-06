# Week 02: Repo Mapping Before Editing

This week teaches the second habit of building a Claude Code-style coding
agent:

```text
map the repo before you change the repo
```

You will not edit files yet. You will inspect the reference checkout, generate a
workspace manifest, identify the main source roots, and write a repo map that a
coding agent could use before planning a change.

Work through this file from top to bottom. Every task appears inside the lesson
at the moment you need it.

## Step 1: Understand The Job Of Week 02

The goal this week is not to memorize every file. The goal is to build a useful
first map of an unfamiliar codebase.

The workflow is:

```text
1. identify the repo roots
2. count the important files
3. name the main modules
4. connect modules to responsibilities
5. find tests and assets
6. write a short repo map before editing
```

For Week 02, the trusted reference is still:

```text
reference/claw-code/
```

You are using the reference to learn what a real coding-agent project has to
notice before it starts changing code.

## Step 2: Learn The Repo Mapping Mental Model

A coding agent should not treat a repo as a flat pile of files.

It should build a map:

```text
entrypoint:
    where the program starts

runtime:
    how a request moves through the system

state:
    where sessions, transcripts, history, or config live

tools:
    where external actions are defined

tests:
    how the repo proves behavior still works

docs:
    where humans explain the system
```

The first useful repo-mapping question is:

```text
If I had to change this behavior, where would I look first?
```

A good map does not answer every future question. It helps you start in the
right neighborhood.

## Step 3: Learn The Words Root, Entrypoint, Module, Surface, And Test

Coding-agent repo inspection uses these words constantly:

- **root** means a top-level directory with a clear job
- **entrypoint** means the file or command where execution begins
- **module** means a grouped piece of source code with one responsibility
- **surface** means the visible commands, tools, APIs, or CLI options
- **test** means executable proof that behavior still works

For now, keep the model simple:

```text
root:
    src, tests, assets, docs, reference

entrypoint:
    the file that receives the command first

module:
    a source file or folder with a named responsibility

surface:
    the commands and tools users or agents can call

test:
    the check you run before trusting a change
```

You are not implementing repo mapping code this week. You are learning what the
map should contain.

## Step 4: Generate The Reference Manifest

Move into the reference checkout:

```bash
cd reference/claw-code
```

Run:

```bash
python -m src.main manifest
```

This command uses:

```text
src/port_manifest.py
```

The manifest answers three basic questions:

```text
where is the source root?
how many Python files exist?
which top-level modules show up?
```

Read the output as a first pass, not as the final truth. A manifest tells you
what exists. You still have to decide what matters.

## Step 5: Inspect The Manifest Builder

Open:

```text
reference/claw-code/src/port_manifest.py
```

Find `build_port_manifest`.

Trace what it does:

```text
1. chooses a source root
2. finds Python files under that root
3. groups files by top-level module or filename
4. counts how many files belong to each group
5. attaches short notes for known important files
6. returns a PortManifest
```

The important idea is that repo mapping starts mechanical:

```text
list files
group files
count files
label obvious responsibilities
```

Then the human or agent adds judgment.

## Step 6: Inspect The Workspace Context

Open:

```text
reference/claw-code/src/context.py
```

Find `build_port_context`.

It names the major roots:

```text
source_root:
    reference/claw-code/src

tests_root:
    reference/claw-code/tests

assets_root:
    reference/claw-code/assets

archive_root:
    reference/claw-code/archive/claude_code_ts_snapshot/src
```

It also counts:

```text
Python files
test files
asset files
whether the archive exists
```

This is a different kind of map from the manifest. The manifest lists modules.
The context names the main repo zones.

## Step 7: Compare Structure And Responsibility

Run:

```bash
python -m src.main summary
```

Pick these files from the output:

```text
src/main.py
src/runtime.py
src/query_engine.py
src/commands.py
src/tools.py
src/transcript.py
src/port_manifest.py
src/context.py
```

Write one sentence for each:

```text
main.py:
runtime.py:
query_engine.py:
commands.py:
tools.py:
transcript.py:
port_manifest.py:
context.py:
```

Example:

```text
main.py: defines the CLI commands that expose the reference workflow.
```

Do not copy long descriptions. A repo map should be short enough to use while
working.

## Step 8: Find The Tests Before Editing

From `reference/claw-code`, run:

```bash
find tests -maxdepth 2 -type f | sort
```

Then run:

```bash
python -m unittest discover -s tests -v
```

If the tests pass, record that. If they fail, record the important failure. A
coding agent needs to know the existing test situation before it changes code.

Read this as a rule:

```text
Before editing, find the test command and learn whether the repo starts green.
```

A failing baseline is not automatically bad. Hiding it is bad.

## Step 9: Create A Change Starting Point

Choose this pretend request:

```text
Add a command that explains the current session transcript.
```

Do not implement it.

Use your repo map to answer:

```text
Where would I inspect first?
Which files might change?
Which test area would I check?
What would be risky to edit without more context?
```

A good first answer might point to:

```text
src/main.py:
    CLI command wiring

src/query_engine.py:
    session and transcript behavior

src/transcript.py:
    transcript storage behavior

tests/:
    baseline checks before and after the change
```

This is what repo mapping gives you: a starting point without pretending you
already know the full solution.

## Step 10: Do The Week 02 Task

Your task is to produce one short repo map:

```text
results/week-02-repo-map.md
```

Run these commands from `reference/claw-code`:

```bash
python -m src.main manifest
python -m src.main summary
find tests -maxdepth 2 -type f | sort
python -m unittest discover -s tests -v
```

Then write the note with this structure:

```markdown
# Week 02 Repo Map

## Commands I Ran

- `python -m src.main manifest`
- `python -m src.main summary`
- `find tests -maxdepth 2 -type f | sort`
- `python -m unittest discover -s tests -v`

## Repo Zones

- source root:
- tests root:
- assets root:
- reference/archive root:

## Important Files

- `src/main.py`:
- `src/runtime.py`:
- `src/query_engine.py`:
- `src/commands.py`:
- `src/tools.py`:
- `src/transcript.py`:
- `src/port_manifest.py`:
- `src/context.py`:

## Test Baseline

Record whether tests passed. If they failed, paste the important error.

## Pretend Change Starting Point

Request:
`Add a command that explains the current session transcript.`

Answer:

- inspect first:
- likely files:
- test area:
- risk:

## My Repo Mapping Mental Model

Write 3-5 sentences in your own words.

## One Thing Still Fuzzy

Write one concept you want to understand better.
```

Keep it short. If you post in Skool, use this format:

```text
Week 02 Submission

Repo zones:

Important files:

Test baseline:

Where I would start for the pretend change:

One thing I want reviewed:
```

## Done Checklist

You are done when:

- `python -m src.main manifest` ran, or you recorded the error
- `python -m src.main summary` ran, or you recorded the error
- you found the test files, or you recorded why you could not
- `python -m unittest discover -s tests -v` ran, or you recorded the error
- your repo map names the main roots and important files in plain language
- your note names one thing you still do not understand

Stop there.
