# Claude Code From Scratch

Build a Claude Code-style coding agent from scratch.

This course teaches the core engineering ideas behind coding agents: tool
calling, repo context, planning, safe file edits, shell commands, test loops,
session memory, CLI workflows, slash commands, and an interactive REPL.

The goal is not to clone every feature of Claude Code. The goal is to build a
clear portfolio project that proves you understand how coding agents work under
the hood.

## What You Will Build

By the end, you will have a small coding-agent framework with a transcript,
tool calling, repo mapping, plan mode, safe file edits, bash/test tools, search,
session resume, slash commands, and an interactive CLI.

## Course Lessons

- `01_intro.md` - first agent turn loop
- `02_repo_mapping.md` - repo context and project summaries
- `03_planning.md` - explicit plan mode
- `04_editing_patches.md` - safe file editing and diffs
- `05_permissions.md` - permission checks before risky actions
- `06_bash_commands.md` - shell command tool
- `07_tool_result_loop.md` - multi-step tool result loop
- `08_testing_loops.md` - running tests and responding to failures
- `09_todo_state.md` - todo state for multi-step work
- `10_search_tools.md` - glob and grep search tools
- `11_read_file_windows.md` - focused file reading
- `12_session_transcript.md` - structured session transcript
- `13_compact_context.md` - context compaction
- `14_end_to_end_build.md` - full runtime orchestration
- `15_cli_sessions.md` - CLI sessions and resume
- `16_slash_commands.md` - local slash commands
- `17_interactive_repl.md` - persistent interactive REPL

## Suggested Build Path

Use the lessons in order. For the Skool challenge, Week 1 is Lessons 1-4:
build the first turn loop, add repo mapping, add plan mode, and implement safe
file edits with diffs. Push your own repo to GitHub and post progress each week.

## Project Structure

Suggested implementation folders: `claudecode/` for agent code, `examples/` for
demos, `tests/` for checks, and `portfolio/` for demo notes or screenshots.

## Quickstart

When your implementation is ready, it should support a flow like:

```bash
python -m pip install -e ".[dev]"
pytest
python examples/demo.py
```

Later lessons add CLI usage like:

```bash
claudecode "explain this repo"
claudecode --resume latest /status
claudecode
```

## Course Principle

The most important rule in this project:

```text
the model chooses tools, but the runtime controls what actually happens
```

That separation is what turns a chatbot into a coding agent.
