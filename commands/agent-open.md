---
description: List conversations waiting on the remote workspace and how to attach to them
allowed-tools: Bash(agent:*)
---

Show me what is currently waiting on the remote agent workspace.

Run both of these:

```
agent ls
agent mcp status
```

Then tell me, briefly:

- which sessions are **idle** (nobody attached) versus in use
- which transferred conversations exist that have not been opened yet

Remind me that attaching is `agent open` in a fresh terminal — it needs a real terminal,
so it cannot be launched from inside this session.

$ARGUMENTS
