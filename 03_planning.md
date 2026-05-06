# Week 03: Build Plan Mode

This week teaches the third habit of building a Claude Code-style coding agent:

```text
planning is an explicit mode for non-trivial work, not a mandatory step before every turn
```

Week 01 built the turn loop:

```text
prompt -> model decision -> validation -> result -> transcript
```

Week 02 added repo context:

```text
repo path -> repo map -> model context -> better model decision
```

Week 03 adds an explicit planning surface:

```text
normal mode -> enter plan mode -> draft plan -> exit plan mode
```

A real coding agent can answer simple questions or inspect files without a
formal plan. But for larger edits, risky changes, or multi-file work, plan mode
slows the agent down and makes its intent reviewable before execution.

Work through this file from top to bottom. Every task appears inside the lesson
at the moment you need it.

## Step 1: Understand The Job Of Week 03

Plan mode is a state.

In normal mode, the agent can do lightweight work:

```text
answer a question
read a file
summarize the repo
run a harmless check
```

In plan mode, the agent should avoid edits and produce a plan:

```text
inspect likely files
name the intended change
name the verification step
wait for approval or exit
```

The Week 03 rule is:

```text
use plan mode when the work is too large or risky to jump straight into execution
```

That is closer to a Claude Code-style workflow than saying every turn must run
through a planner.

## Step 2: Define Agent Modes

Start with a small mode type:

```python
from dataclasses import dataclass
from enum import Enum
```

`Enum` gives us named states:

```python
class AgentMode(str, Enum):
    NORMAL = "normal"
    PLAN = "plan"
```

Why use an enum instead of plain strings?

```text
plain strings:
    easy to mistype

enum values:
    constrained to the modes the runtime understands
```

The first two modes are enough:

```text
normal:
    regular agent behavior

plan:
    planning-only behavior before edits
```

## Step 3: Define Plan Data

Plan mode needs a structured output.

First define one step:

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

Now define the full plan:

```python
@dataclass(frozen=True)
class AgentPlan:
    request: str
    summary: str
    steps: tuple[PlanStep, ...]
```

The plan keeps:

```text
request:
    the user's original task

summary:
    one sentence describing the plan

steps:
    ordered work items the user can review
```

## Step 4: Define Plan Mode State

The runtime needs to remember whether it is currently planning.

```python
@dataclass
class AgentState:
    mode: AgentMode = AgentMode.NORMAL
    active_plan: AgentPlan | None = None
```

Break it down:

```text
mode:
    tells the runtime how to behave on the next turn

active_plan:
    stores the current draft plan, if one exists
```

This is a small but important shift from Week 01:

```text
Week 01 stored transcript state.
Week 03 stores mode state.
```

Mode state lets the agent behave differently without changing the whole runtime.

## Step 5: Decide When To Suggest Plan Mode

Do not enter plan mode for everything.

Use a tiny heuristic:

```python
def should_enter_plan_mode(request: str) -> bool:
    lowered = request.lower()
```

Start by normalizing the request:

```text
lowered:
    lets "Refactor" and "refactor" match the same way
```

Now detect non-trivial work:

```python
    planning_words = (
        "refactor",
        "redesign",
        "migrate",
        "multi-file",
        "architecture",
        "from scratch",
    )
    return any(word in lowered for word in planning_words)
```

This heuristic is intentionally simple. In a real agent, the model can decide
when plan mode is useful from the prompt, repo context, and transcript.

The runtime still needs a simple policy:

```text
small request:
    normal mode is fine

large or risky request:
    suggest or enter plan mode
```

## Step 6: Choose Inspection Targets From Repo Context

Plan mode should use the `RepoMap` from Week 02.

```python
from .repo_map import RepoMap
```

Write a helper that picks likely files:

```python
def choose_plan_targets(repo_map: RepoMap) -> tuple[str, ...]:
    targets: list[str] = []
    targets.extend(repo_map.entrypoint_candidates[:3])
    targets.extend(repo_map.test_candidates[:3])
```

This starts with:

```text
entrypoint candidates:
    where behavior may begin

test candidates:
    where verification may happen
```

Add metadata files:

```python
    for name in ("README.md", "pyproject.toml", "package.json"):
        if name in repo_map.top_level_files:
            targets.append(name)
```

Remove duplicates:

```python
    return tuple(dict.fromkeys(targets))
```

Why this matters:

```text
plan mode should propose a small inspection set before proposing edits
```

The model can later refine this list. The runtime gives it a useful starting
point.

## Step 7: Draft A Plan

Now build the plan.

```python
def draft_plan(request: str, repo_map: RepoMap) -> AgentPlan:
    targets = choose_plan_targets(repo_map)
    steps: list[PlanStep] = []
```

Start with inspection steps:

```python
    for target in targets:
        steps.append(
            PlanStep(
                action="inspect",
                target=target,
                reason="Understand the relevant code before execution.",
            )
        )
```

Then add the intended edit step:

```python
    steps.append(
        PlanStep(
            action="edit",
            target="smallest relevant source file",
            reason="Keep the change narrow and reviewable.",
        )
    )
```

Then add verification:

```python
    steps.append(
        PlanStep(
            action="test",
            target="focused test command",
            reason="Verify the behavior before reporting success.",
        )
    )
```

Return the plan:

```python
    return AgentPlan(
        request=request,
        summary=f"Plan mode draft with {len(steps)} steps.",
        steps=tuple(steps),
    )
```

This is not the agent editing. This is the agent preparing a reviewable plan.

## Step 8: Render The Plan

Humans need readable output.

```python
def render_plan(plan: AgentPlan) -> str:
    lines = [
        "# Plan Mode Draft",
        "",
        f"Request: {plan.request}",
        f"Summary: {plan.summary}",
        "",
        "## Steps",
    ]
```

Now add numbered steps:

```python
    for index, step in enumerate(plan.steps, start=1):
        lines.append(
            f"{index}. {step.action}: {step.target} - {step.reason}"
        )
```

Why numbered steps?

```text
the user can approve, reject, or discuss a specific step
```

Finish:

```python
    return "\n".join(lines)
```

This renderer is what the agent can append to the transcript and show to the
user while in plan mode.

## Step 9: Enter And Exit Plan Mode

Add two runtime helpers:

```python
def enter_plan_mode(state: AgentState, request: str, repo_map: RepoMap) -> AgentPlan:
    plan = draft_plan(request, repo_map)
    state.mode = AgentMode.PLAN
    state.active_plan = plan
    return plan
```

Line by line:

```text
draft_plan:
    creates the proposed plan

state.mode = AgentMode.PLAN:
    changes runtime behavior

state.active_plan = plan:
    stores the plan for the next turn
```

Now exit:

```python
def exit_plan_mode(state: AgentState) -> AgentPlan | None:
    plan = state.active_plan
    state.mode = AgentMode.NORMAL
    state.active_plan = None
    return plan
```

Exiting plan mode does not execute the plan yet. It simply returns the accepted
or abandoned draft and switches back to normal mode.

The course will execute real edits later.

## Step 10: Connect Plan Mode To The Agent Loop

Week 01 had:

```text
prompt -> model decision -> validation -> result -> transcript
```

Week 02 added:

```text
repo context
```

Week 03 adds:

```text
mode state
```

In a future `CodingAgent.run_turn`, the runtime can do:

```python
if should_enter_plan_mode(prompt):
    repo_map = build_repo_map(".")
    plan = enter_plan_mode(self.state, prompt, repo_map)
    self.transcript.append("plan", render_plan(plan))
    return
```

Normal mode still exists. A tiny request does not need a formal plan.

Plan mode is for work where the user should see intent before execution:

```text
refactors
migrations
architecture changes
multi-file edits
ambiguous implementation requests
```

## Step 11: Add A Tiny CLI

Give plan mode a command-line demo:

```python
def main() -> int:
    import argparse
    from .repo_map import build_repo_map

    parser = argparse.ArgumentParser()
    parser.add_argument("request")
    parser.add_argument("--path", default=".")
    args = parser.parse_args()

    state = AgentState()
    repo_map = build_repo_map(args.path)
    plan = enter_plan_mode(state, args.request, repo_map)
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
python -m claudecode.plan_mode "refactor the CLI"
```

The command should print a plan mode draft and make no edits.

## Step 12: The Week 03 Build

Your build this week is one small file:

```text
claudecode/plan_mode.py
```

It should contain:

```text
AgentMode
PlanStep
AgentPlan
AgentState
should_enter_plan_mode
choose_plan_targets
draft_plan
render_plan
enter_plan_mode
exit_plan_mode
main
```

Optional tiny smoke test:

```python
from pathlib import Path

from claudecode.plan_mode import AgentMode, AgentState, enter_plan_mode, exit_plan_mode
from claudecode.repo_map import RepoMap


def test_plan_mode_enters_and_exits():
    state = AgentState()
    repo_map = RepoMap(
        root=Path("."),
        top_level_dirs=("claudecode", "tests"),
        top_level_files=("README.md",),
        entrypoint_candidates=("claudecode/cli.py",),
        test_candidates=("tests/test_cli.py",),
        total_files=3,
    )

    plan = enter_plan_mode(state, "refactor the CLI", repo_map)

    assert state.mode == AgentMode.PLAN
    assert state.active_plan == plan

    exited = exit_plan_mode(state)

    assert exited == plan
    assert state.mode == AgentMode.NORMAL
    assert state.active_plan is None
```

This test proves the important behavior:

```text
plan mode is stateful and reversible
```

## Step 13: Done Checklist

You are done when:

- `AgentMode` has `NORMAL` and `PLAN`
- `AgentState` stores the current mode and active plan
- `should_enter_plan_mode` identifies larger requests
- `draft_plan` creates inspection, edit, and test steps
- `render_plan` produces readable numbered output
- `enter_plan_mode` stores the plan and switches modes
- `exit_plan_mode` clears the plan and returns to normal mode
- `python -m claudecode.plan_mode "refactor the CLI"` prints a plan draft
- you can explain when explicit planning should and should not be used

Stop there.

## Skool Submission

Use this format:

```text
Week 03 Submission

My plan mode in one sentence:

Example request:

Why this request should or should not enter plan mode:

First planned inspection:

Exit behavior:

How this connects to Week 01 and Week 02:

One thing I want reviewed:
```
