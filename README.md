# Claude Code From Scratch

A practical course and codebase for building a Claude Code-style coding agent
from scratch: repo inspection, planning, file edits, test runs, patch review,
retry loops, and shipping workflows.

Creator note: lessons are informed by real coding-agent systems, but learners
build the project from scratch. No external repo is required.

## Portfolio Outcome

- inspect a real repo and build a reliable repo map
- turn user requests into concrete implementation plans
- edit code in small, reviewable patches
- run tests and respond to failures with an agent loop
- package the workflow as a portfolio project and a Skool challenge

## Course Shape

- `01_intro.md`
- `02_repo_mapping.md`
- `03_planning.md`
- `04_editing_patches.md`
- `05_testing_loops.md`
- `06_review_and_retry.md`
- `07_safe_tool_use.md`
- `08_terminal_workflows.md`
- `09_end_to_end_build.md`
- `10_portfolio_packaging.md`

## Structure

- `claudecode/` contains the assistant workflow code and CLI.
- `examples/` contains runnable demos.
- `tests/` contains correctness checks.
- `portfolio/` contains interview and resume assets.
- `references/` tracks public sources and scope boundaries.

## Quickstart

```bash
python -m pip install -e ".[dev]"
pytest
python examples/demo.py
```

## Positioning

This project is the closed, opinionated sibling to OpenClaw. It focuses on a
Claude Code-style developer workflow, with the course built around inspecting a
repo, planning a change, patching it, testing it, and shipping it cleanly.
