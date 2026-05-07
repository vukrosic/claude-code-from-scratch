# Claude Code From Scratch

Build a Claude Code-style coding agent from scratch.

This course teaches the core engineering ideas behind coding agents: tool
calling, repo context, planning, safe file edits, shell commands, test loops,
session memory, CLI workflows, slash commands, and an interactive REPL.

The goal is not to clone every feature of Claude Code. The goal is to build a
clear portfolio project that proves you understand how coding agents work under
the hood.

## What You Will Build

By the end, you will have a small coding-agent framework that can:

- keep a transcript of user, assistant, tool call, and tool result messages
- expose tools with names, descriptions, and input schemas
- let the model request tools while the runtime controls execution
- scan a repo and summarize useful project context
- draft simple plans for larger changes
- safely edit files with exact string replacement and reviewable diffs
- run shell commands and tests through a controlled tool layer
- keep todo state across multi-step work
- search files, grep code, and read focused file windows
- save and resume sessions from JSONL files
- compact old context when the transcript gets too large
- run as a CLI with `/status`, `/diff`, `/session list`, and `/resume`
- keep an interactive REPL session alive across many turns

## What You Will Learn

This project teaches practical AI engineering and software engineering skills:

- how LLM tool calling actually fits into an agent runtime
- why the model proposes actions but the runtime validates and executes them
- how to represent messages, tool calls, tool results, and session history
- how to give the model repo context without flooding the prompt
- how coding agents search before editing
- how safe file editing works with `old_string` and `new_string`
- how to run tests, inspect failures, and retry with new context
- how CLI agents manage sessions, slash commands, and resume workflows
- how to design small modules that are easy to test
- how to turn an AI project into a portfolio-ready GitHub repo

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

## Portfolio Outcome

A finished version of this project should include:

- a public GitHub repo
- a working demo of the agent loop
- tests for the core runtime and tools
- a README explaining the architecture
- screenshots or terminal output showing the agent using tools
- a short demo video or write-up explaining what you built

This is meant to be shown as a career portfolio project for AI engineering,
LLM engineering, coding-agent work, and software engineering roles.

## Suggested Build Path

If you are doing the Skool challenge, use the lessons in order.

Week 1 starts with Lessons 1-4:

- build the first agent turn loop
- add repo mapping
- add plan mode
- add safe file editing with diffs

Push your own repo to GitHub and post your progress each week.

## Project Structure

Suggested structure for your implementation:

- `claudecode/` contains the assistant workflow code and CLI
- `examples/` contains runnable demos
- `tests/` contains correctness checks
- `portfolio/` contains demo notes, screenshots, and resume assets
- `reference/` can hold private/local notes, but learners do not need an external repo

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
