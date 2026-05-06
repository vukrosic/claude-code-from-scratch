# Week 05: Permission-Gated Tools

This week adds the runtime guard that sits in front of tool execution.

```text
model requests tool -> runtime checks permission -> tool runs or gets blocked
```

This is how file editing becomes safer: `edit_file` may exist, but it does not
run in read-only mode.

## Step 1: Define Permission Modes

Use ordered modes:

```python
from dataclasses import dataclass
from enum import IntEnum


class PermissionMode(IntEnum):
    READ_ONLY = 1
    WORKSPACE_WRITE = 2
    DANGER_FULL_ACCESS = 3
```

The order is the behavior:

```text
READ_ONLY < WORKSPACE_WRITE < DANGER_FULL_ACCESS
```

So `DANGER_FULL_ACCESS` can run anything, `WORKSPACE_WRITE` can run read/write
workspace tools, and `READ_ONLY` can only run read tools.

Full coding-agent runtimes may also have prompt/allow modes. This lesson starts
with the three non-interactive modes because they are the core gate.

## Step 2: Put Permissions On Tool Specs

Every tool should declare the permission it needs.

```python
@dataclass(frozen=True)
class ToolSpec:
    name: str
    description: str
    required_permission: PermissionMode
```

Define the tools we have so far:

```python
TOOL_SPECS = {
    "read_file": ToolSpec(
        "read_file",
        "Read a text file from the workspace.",
        PermissionMode.READ_ONLY,
    ),
    "write_file": ToolSpec(
        "write_file",
        "Write a text file in the workspace.",
        PermissionMode.WORKSPACE_WRITE,
    ),
    "edit_file": ToolSpec(
        "edit_file",
        "Replace text in a workspace file.",
        PermissionMode.WORKSPACE_WRITE,
    ),
    "bash": ToolSpec(
        "bash",
        "Run a shell command.",
        PermissionMode.DANGER_FULL_ACCESS,
    ),
}
```

This is the important map:

```text
read_file  -> read-only
write_file -> workspace-write
edit_file  -> workspace-write
bash       -> danger-full-access
```

Bash can later become smarter by classifying commands. For now, keep it
conservative.

## Step 3: Build A Permission Policy

The policy stores the current mode and the requirement for each tool.

```python
@dataclass(frozen=True)
class PermissionPolicy:
    active_mode: PermissionMode
    tool_requirements: dict[str, PermissionMode]
```

```python
def build_permission_policy(active_mode: PermissionMode) -> PermissionPolicy:
    return PermissionPolicy(
        active_mode=active_mode,
        tool_requirements={
            name: spec.required_permission
            for name, spec in TOOL_SPECS.items()
        },
    )
```

Now the runtime can answer:

```text
what mode am I in?
what mode does this tool require?
```

## Step 4: Return A Structured Decision

Do not return only `True` or `False`. Keep the denial reason.

```python
@dataclass(frozen=True)
class EnforcementResult:
    allowed: bool
    tool: str
    active_mode: PermissionMode
    required_mode: PermissionMode
    reason: str
```

The reason becomes the error shown to the model and user.

## Step 5: Build The Enforcer

The enforcer compares active mode to required mode.

```python
class PermissionEnforcer:
    def __init__(self, policy: PermissionPolicy):
        self.policy = policy

    def required_mode_for(self, tool_name: str) -> PermissionMode:
        return self.policy.tool_requirements.get(
            tool_name,
            PermissionMode.DANGER_FULL_ACCESS,
        )
```

Unknown tools default to `DANGER_FULL_ACCESS`. Unknown should mean risky, not
free.

Now add the check:

```python
    def check(self, tool_name: str) -> EnforcementResult:
        active = self.policy.active_mode
        required = self.required_mode_for(tool_name)

        if active >= required:
            return EnforcementResult(
                allowed=True,
                tool=tool_name,
                active_mode=active,
                required_mode=required,
                reason="allowed",
            )

        return EnforcementResult(
            allowed=False,
            tool=tool_name,
            active_mode=active,
            required_mode=required,
            reason=(
                f"tool '{tool_name}' requires {required.name}; "
                f"current mode is {active.name}"
            ),
        )
```

That is the permission gate.

## Step 6: Check Before Dispatch

Tool dispatch should start with permission enforcement.

```python
def execute_tool(
    name: str,
    arguments: dict,
    enforcer: PermissionEnforcer,
) -> str:
    decision = enforcer.check(name)
    if not decision.allowed:
        return decision.reason
```

Only after that should the runtime call the tool:

```python
    if name == "read_file":
        return f"reading {arguments['path']}"

    if name == "write_file":
        return f"writing {arguments['path']}"

    if name == "edit_file":
        return f"editing {arguments['path']}"

    if name == "bash":
        return f"running {arguments['command']}"

    return f"unsupported tool: {name}"
```

The fake tool bodies are placeholders. The real lesson is the ordering:

```text
permission first
tool body second
```

## Step 7: Quick Mode Check

Read-only mode:

```python
policy = build_permission_policy(PermissionMode.READ_ONLY)
enforcer = PermissionEnforcer(policy)

print(execute_tool("read_file", {"path": "README.md"}, enforcer))
print(execute_tool("edit_file", {"path": "README.md"}, enforcer))
```

Expected:

```text
read_file runs
edit_file is blocked
```

Workspace-write mode:

```python
policy = build_permission_policy(PermissionMode.WORKSPACE_WRITE)
enforcer = PermissionEnforcer(policy)

print(execute_tool("edit_file", {"path": "README.md"}, enforcer))
print(execute_tool("bash", {"command": "rm -rf /tmp/demo"}, enforcer))
```

Expected:

```text
edit_file runs
bash is blocked
```

## Step 8: Test The Gate

Create:

```text
claudecode/permissions.py
```

Add these tests:

```python
from claudecode.permissions import (
    PermissionEnforcer,
    PermissionMode,
    build_permission_policy,
)
```

```python
def test_read_only_allows_read_file():
    enforcer = PermissionEnforcer(
        build_permission_policy(PermissionMode.READ_ONLY)
    )

    assert enforcer.check("read_file").allowed is True
```

```python
def test_read_only_blocks_edit_file():
    enforcer = PermissionEnforcer(
        build_permission_policy(PermissionMode.READ_ONLY)
    )

    result = enforcer.check("edit_file")

    assert result.allowed is False
    assert result.required_mode == PermissionMode.WORKSPACE_WRITE
```

```python
def test_workspace_write_blocks_bash():
    enforcer = PermissionEnforcer(
        build_permission_policy(PermissionMode.WORKSPACE_WRITE)
    )

    result = enforcer.check("bash")

    assert result.allowed is False
    assert result.required_mode == PermissionMode.DANGER_FULL_ACCESS
```

```python
def test_danger_full_access_allows_bash():
    enforcer = PermissionEnforcer(
        build_permission_policy(PermissionMode.DANGER_FULL_ACCESS)
    )

    assert enforcer.check("bash").allowed is True
```

Stop when these are true:

- every tool has a required permission
- read-only mode blocks `edit_file` and `write_file`
- workspace-write mode allows file tools
- workspace-write mode blocks `bash`
- danger-full-access allows `bash`
- dispatch checks permission before running the tool body

## Skool Submission

Use this format:

```text
Week 05 Submission

My permission gate in one sentence:

Tool permission table:

Read-only blocked example:

Workspace-write allowed example:

Why permission checks happen before dispatch:
```
