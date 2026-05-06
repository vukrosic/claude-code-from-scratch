# Week 03: Build A Plan Before Tool Execution

This week teaches the third habit of building a Claude Code-style coding agent:

```text
the agent should plan before it edits
```

Week 01 built the turn loop:

```text
prompt -> model decision -> validation -> result -> transcript
```

Week 02 added repo context:

```text
repo path -> repo map -> model context -> better model decision
```

Week 03 adds the next missing layer:

```text
prompt + repo map -> plan -> tool calls
```

You will build a tiny planner. It will turn a user request and a `RepoMap` into
a short list of inspection, edit, and verification steps. It will not edit files
yet. Planning is the bridge between "the model wants to do something" and "the
runtime is allowed to touch the repo."

Work through this file from top to bottom. Every task appears inside the lesson
at the moment you need it.

## Step 1: Understand The Job Of Week 03

A weak coding agent jumps from request to edit:

```text
user: add a CLI command
agent: edits random file
```

A stronger coding agent creates a plan first:

```text
1. inspect likely CLI entrypoint
2. inspect tests
3. add the command in one small file
4. add or update one focused test
5. run the test command
```

This is not bureaucracy. It is how the agent makes its work reviewable.

The Week 03 rule is:

```text
do not execute an edit until the plan names what will be inspected, changed, and verified
```

## Step 2: Learn The Planning Shape

Start with three small objects:

```python
from dataclasses import dataclass
```

The first object is one step:

```python
@dataclass(frozen=True)
class PlanStep:
    action: str
    target: str
    reason: str
```

Read the fields as:

```text
action:
    inspect, edit, test, or report

target:
    the file, command, or area the step is about

reason:
    why this step belongs in the plan
```

The second object is the full plan:

```python
@dataclass(frozen=True)
class AgentPlan:
    request: str
    summary: str
    steps: tuple[PlanStep, ...]
```

The plan keeps the original request, a one-sentence summary, and ordered steps.

The third object describes the kind of work:

```python
@dataclass(frozen=True)
class RequestKind:
    name: str
    confidence: float
```

This lets the agent say:

```text
I think this is a feature request, but I am not 100 percent sure.
```

Confidence should make the plan more honest, not more complicated.

## Step 3: Classify The Request

The real model can classify requests semantically. For the from-scratch build,
use a tiny deterministic classifier so the planner is runnable.

```python
def classify_request(request: str) -> RequestKind:
    lowered = request.lower()
```

Start by normalizing the request:

```text
lowered:
    makes "Add", "add", and "ADD" behave the same
```

Now add a few simple categories:

```python
    if any(word in lowered for word in ("add", "create", "implement")):
        return RequestKind("feature", 0.7)

    if any(word in lowered for word in ("fix", "bug", "broken", "failing")):
        return RequestKind("bugfix", 0.7)

    if any(word in lowered for word in ("explain", "summarize", "document")):
        return RequestKind("explanation", 0.7)

    return RequestKind("unknown", 0.3)
```

This is not replacing the LLM. It is a teaching scaffold.

The future architecture is:

```text
LLM classifies intent from prompt + repo context + transcript
runtime stores the classification in a structured plan
```

Week 03 uses a small function so you can build the runtime shape first.

## Step 4: Pick Files To Inspect

The planner should use the `RepoMap` from Week 02.

Import it:

```python
from .repo_map import RepoMap
```

Now write a helper that chooses inspection targets:

```python
def choose_inspection_targets(repo_map: RepoMap) -> tuple[str, ...]:
    targets: list[str] = []
```

Start with likely entrypoints:

```python
    targets.extend(repo_map.entrypoint_candidates[:3])
```

Why only three?

```text
plans should stay small enough to follow
```

Then add tests:

```python
    targets.extend(repo_map.test_candidates[:3])
```

Finally, use top-level metadata files:

```python
    for name in ("README.md", "pyproject.toml", "package.json"):
        if name in repo_map.top_level_files:
            targets.append(name)
```

Remove duplicates while preserving order:

```python
    return tuple(dict.fromkeys(targets))
```

That last line is a compact Python trick:

```text
dict.fromkeys(targets):
    keeps the first copy of each value

tuple(...):
    returns an immutable snapshot
```

The goal is not perfect file selection. The goal is to give the model a small,
reasonable first inspection list.

## Step 5: Build The First Plan Steps

Planning should always start with inspection.

```python
def build_inspection_steps(targets: tuple[str, ...]) -> list[PlanStep]:
    steps: list[PlanStep] = []

    for target in targets:
        steps.append(
            PlanStep(
                action="inspect",
                target=target,
                reason="Understand the relevant code before editing.",
            )
        )

    return steps
```

This loop says:

```text
for each likely target
create one inspect step
explain why it exists
```

The reason field matters. A coding agent should be able to explain why it wants
to open a file.

## Step 6: Add Intent-Specific Steps

Now add steps based on the request kind.

```python
def build_intent_steps(kind: RequestKind) -> list[PlanStep]:
    if kind.name == "feature":
        return [
            PlanStep(
                action="edit",
                target="smallest relevant source file",
                reason="Implement the requested behavior in the narrowest place.",
            ),
            PlanStep(
                action="test",
                target="focused test command",
                reason="Verify the new behavior works.",
            ),
        ]
```

For bug fixes:

```python
    if kind.name == "bugfix":
        return [
            PlanStep(
                action="inspect",
                target="failing test or error output",
                reason="Reproduce or understand the failure before editing.",
            ),
            PlanStep(
                action="edit",
                target="smallest file causing the failure",
                reason="Fix the behavior with minimal blast radius.",
            ),
            PlanStep(
                action="test",
                target="failing test command",
                reason="Confirm the fix addresses the original failure.",
            ),
        ]
```

For explanation requests:

```python
    if kind.name == "explanation":
        return [
            PlanStep(
                action="report",
                target="plain-language answer",
                reason="The request may not require code changes.",
            )
        ]
```

Fallback:

```python
    return [
        PlanStep(
            action="report",
            target="clarifying summary",
            reason="The request is too ambiguous for an edit plan.",
        )
    ]
```

This structure mirrors how serious coding agents behave:

```text
feature:
    inspect -> edit -> test

bugfix:
    reproduce -> edit -> test

explanation:
    inspect -> report

unknown:
    clarify before editing
```

## Step 7: Build The Full Plan

Now combine request classification, repo context, and steps:

```python
def build_plan(request: str, repo_map: RepoMap) -> AgentPlan:
    kind = classify_request(request)
    targets = choose_inspection_targets(repo_map)
    steps = [
        *build_inspection_steps(targets),
        *build_intent_steps(kind),
    ]
```

Break it down:

```text
kind:
    what type of request this seems to be

targets:
    likely files to inspect from the repo map

steps:
    inspection steps first, then intent-specific steps
```

Now return the plan:

```python
    return AgentPlan(
        request=request,
        summary=f"{kind.name} request with {len(steps)} planned steps",
        steps=tuple(steps),
    )
```

This is the Week 03 product:

```text
request + repo map -> structured plan
```

## Step 8: Render The Plan

The model and runtime need structured data. Humans need readable output.

```python
def render_plan(plan: AgentPlan) -> str:
    lines = [
        "# Agent Plan",
        "",
        f"Request: {plan.request}",
        f"Summary: {plan.summary}",
        "",
        "## Steps",
    ]
```

Then list the steps:

```python
    for index, step in enumerate(plan.steps, start=1):
        lines.append(
            f"{index}. {step.action}: {step.target} - {step.reason}"
        )
```

Why `enumerate(..., start=1)`?

```text
plans are easier to review when steps are numbered from 1
```

Finish:

```python
    return "\n".join(lines)
```

This renderer will eventually be appended to the transcript before edits begin.

## Step 9: Connect Planning To The Existing Loop

Week 01 had:

```text
prompt -> model decision -> validation -> result -> transcript
```

Week 02 added:

```text
repo map -> model context
```

Week 03 adds:

```text
model context + prompt -> plan
```

In a future `CodingAgent.run_turn`, the runtime can do:

```python
repo_map = build_repo_map(".")
plan = build_plan(prompt, repo_map)
self.transcript.append("plan", render_plan(plan))
```

Then the model can decide the next tool call with:

```text
prompt
tool specs
repo context
plan
transcript
```

This is the natural order:

```text
map first
plan second
edit third
test fourth
```

Week 03 stops at planning.

## Step 10: Add A Tiny CLI

Give the planner a command-line entrypoint:

```python
def main() -> int:
    import argparse
    from .repo_map import build_repo_map

    parser = argparse.ArgumentParser()
    parser.add_argument("request")
    parser.add_argument("--path", default=".")
    args = parser.parse_args()

    repo_map = build_repo_map(args.path)
    plan = build_plan(args.request, repo_map)
    print(render_plan(plan))
    return 0
```

Finish the module:

```python
if __name__ == "__main__":
    raise SystemExit(main())
```

Now the learner can run:

```bash
python -m claudecode.planning "add a repo map command"
```

The command should print a short plan, not edit files.

## Step 11: The Week 03 Build

Your build this week is one small file:

```text
claudecode/planning.py
```

It should contain:

```text
PlanStep
AgentPlan
RequestKind
classify_request
choose_inspection_targets
build_inspection_steps
build_intent_steps
build_plan
render_plan
main
```

Optional tiny smoke test:

```python
from pathlib import Path

from claudecode.planning import build_plan
from claudecode.repo_map import RepoMap


def test_feature_plan_starts_with_inspection():
    repo_map = RepoMap(
        root=Path("."),
        top_level_dirs=("claudecode", "tests"),
        top_level_files=("README.md",),
        entrypoint_candidates=("claudecode/cli.py",),
        test_candidates=("tests/test_cli.py",),
        total_files=3,
    )

    plan = build_plan("add a CLI command", repo_map)

    assert plan.steps[0].action == "inspect"
    assert any(step.action == "edit" for step in plan.steps)
    assert any(step.action == "test" for step in plan.steps)
```

This test proves the important behavior:

```text
feature requests inspect before edit and test
```

## Step 12: Done Checklist

You are done when:

- `classify_request` returns a `RequestKind`
- `choose_inspection_targets` uses `RepoMap`
- `build_plan` creates inspection steps before edit steps
- feature requests include edit and test steps
- bugfix requests include failure inspection and test steps
- explanation requests can avoid edits
- `render_plan` produces readable numbered output
- `python -m claudecode.planning "add a CLI command"` prints a plan
- you can explain why planning comes before editing

Stop there.

## Skool Submission

Use this format:

```text
Week 03 Submission

My planner in one sentence:

Example request:

First inspection target:

Planned edit step:

Planned test step:

How this connects to Week 01 and Week 02:

One thing I want reviewed:
```
