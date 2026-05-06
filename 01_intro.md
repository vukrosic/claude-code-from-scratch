# Week 01: Coding Agent Turn Loop And Baseline

This week teaches the first habit of building a Claude Code-style coding agent:

```text
understand the agent loop before you trust the agent to edit code
```

You will not build file editing yet. You will inspect the reference checkout,
run the mirrored agent workflow, understand what happens during one prompt, and
write your first baseline note.

Work through this file from top to bottom. Every task appears inside the lesson
at the moment you need it.

## Step 1: Understand The Job Of Week 01

The goal this week is not to clone all of Claude Code. The goal is to understand
the smallest useful loop behind a coding agent.

The workflow is:

```text
1. receive a user request
2. inspect the current repo context
3. choose relevant commands and tools
4. run one agent turn
5. store the transcript
6. report what happened
```

For Week 01, the trusted reference is the local checkout in:

```text
reference/claw-code/
```

That repo is not the course implementation. It is the source shape we inspect
before building our own smaller version.

## Step 2: Learn The Coding Agent Mental Model

A normal chatbot answers from conversation context. A coding agent has to work
inside a repo.

That means every useful turn needs more than a prompt:

```text
user request:
    what the human wants

repo context:
    files, project shape, tests, current changes

tool surface:
    actions the agent is allowed to take

transcript:
    what already happened in this session

result:
    the answer, patch, test result, or next decision
```

The first useful coding-agent question is:

```text
What does the agent need to know before it edits anything?
```

If you skip that question, the agent starts making random changes. If you answer
it well, the agent becomes much easier to trust.

## Step 3: Learn The Words Prompt, Command, Tool, Turn, And Transcript

Claude Code-style systems use these ideas constantly:

- **prompt** means the user's request for this turn
- **command** means a named workflow the agent can route to
- **tool** means an external action like reading files or running shell commands
- **turn** means one cycle of request, routing, action, and response
- **transcript** means the stored history of the session

For now, keep the model simple:

```text
prompt:
    "read README and explain the repo"

command:
    a higher-level workflow the agent may choose

tool:
    a concrete capability the agent may use

turn:
    one pass through the agent runtime

transcript:
    the memory trail that lets later turns know what happened
```

You are not building commands or tools this week. You are learning how they fit
into one agent turn.

## Step 4: Inspect The Reference Surface

Move into the reference checkout:

```bash
cd reference/claw-code
```

Run the summary command:

```bash
python -m src.main summary
```

This prints the mirrored workspace shape. Read the output as a map:

```text
main.py:
    CLI entrypoint

runtime.py:
    routes prompts and bootstraps sessions

query_engine.py:
    stores turns, usage, stop reasons, and transcript state

commands.py:
    loads the command inventory

tools.py:
    loads the tool inventory
```

This is the first baseline. Before building your own agent, you should know
which pieces a real reference keeps separate.

## Step 5: Inspect The CLI Entrypoint

Open:

```text
reference/claw-code/src/main.py
```

Find the parser commands:

```text
summary
commands
tools
route
bootstrap
turn-loop
flush-transcript
load-session
```

Read them as product features:

```text
summary:
    show what the workspace contains

route:
    match a prompt to commands and tools

turn-loop:
    simulate multiple agent turns

flush-transcript:
    persist session memory
```

The lesson is not that you need the exact same commands. The lesson is that a
coding agent needs a visible control surface. If you cannot run and inspect the
loop, you cannot debug it.

## Step 6: Route One Prompt

Run:

```bash
python -m src.main route "read README and explain the repo"
```

The output should list matched commands and tools. The exact matches can change
as the reference evolves, but the shape should look like this:

```text
command    some-command    score    source-path
tool       some-tool       score    source-path
```

Read the result as a sentence:

```text
Given this prompt, the runtime found these command and tool candidates.
```

This is not an LLM response yet. It is routing. Routing decides what parts of
the agent surface are relevant before a turn runs.

## Step 7: Run A Tiny Turn Loop

Run:

```bash
python -m src.main turn-loop "read README and explain the repo" --max-turns 2
```

The reference loop prints one section per turn:

```text
## Turn 1
Prompt: read README and explain the repo
Matched commands: ...
Matched tools: ...
Permission denials: ...
stop_reason=completed
```

Read this as the first skeleton of a coding agent:

```text
prompt in
route commands and tools
record permission state
produce a turn result
decide why the turn stopped
```

Later, your implementation will replace the mirror output with real file reads,
patches, tests, and repair loops. Week 01 is only about seeing the skeleton.

## Step 8: Inspect The Turn Engine

Open:

```text
reference/claw-code/src/query_engine.py
```

Find `QueryEnginePort.submit_message`.

Trace what it does:

```text
1. checks whether max turns were reached
2. formats a response summary
3. updates token usage
4. appends the prompt to mutable messages
5. appends the prompt to the transcript store
6. compacts messages if needed
7. returns a TurnResult
```

The important idea is that a turn has state. It is not just a function that
takes a string and returns a string. It updates memory, usage, permissions, and
the stop reason.

## Step 9: Inspect Transcript Storage

Open:

```text
reference/claw-code/src/transcript.py
```

The transcript store is intentionally small:

```text
append:
    add a new entry

compact:
    keep only recent entries

replay:
    return stored entries

flush:
    mark transcript as persisted
```

This is the second useful coding-agent question:

```text
What should the next turn remember from this turn?
```

Without a transcript, every turn starts from zero. With a transcript, the agent
can build continuity.

## Step 10: Do The Week 01 Task

Your task is to produce one short baseline note:

```text
results/week-01-baseline.md
```

Run these commands from `reference/claw-code`:

```bash
python -m src.main summary
python -m src.main route "read README and explain the repo"
python -m src.main turn-loop "read README and explain the repo" --max-turns 2
```

Then write the note with this structure:

```markdown
# Week 01 Baseline

## Reference Commands I Ran

- `python -m src.main summary`
- `python -m src.main route "read README and explain the repo"`
- `python -m src.main turn-loop "read README and explain the repo" --max-turns 2`

## Reference Surface

Write one sentence each:

- `src/main.py`:
- `src/runtime.py`:
- `src/query_engine.py`:
- `src/commands.py`:
- `src/tools.py`:
- `src/transcript.py`:

## Prompt Routing

Paste or summarize the command/tool matches from `route`.

## Turn Loop

Answer these questions:

- What goes into one turn?
- What comes out of one turn?
- What does `stop_reason=completed` mean?

## My Coding Agent Mental Model

Write 3-5 sentences in your own words.

## One Thing Still Fuzzy

Write one concept you want to understand better.
```

Keep it short. If you post in Skool, use this format:

```text
Week 01 Submission

Reference commands:

My explanation of the coding-agent turn loop:

What routing does:

Why transcripts matter:

One thing I want reviewed:
```

## Done Checklist

You are done when:

- `python -m src.main summary` ran, or you recorded the error
- `python -m src.main route "read README and explain the repo"` ran, or you recorded the error
- `python -m src.main turn-loop "read README and explain the repo" --max-turns 2` ran, or you recorded the error
- your note explains prompt, command, tool, turn, and transcript in plain language
- your note names one thing you still do not understand

Stop there.
