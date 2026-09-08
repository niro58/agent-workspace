---
description: Move this conversation to the remote workspace, resumed and ready to continue
allowed-tools: Bash(agent:*), Bash(git:*)
---

Move the conversation we are in right now to the remote agent workspace, so it can be
continued there after this machine is off.

Run exactly this, from the current working directory:

```
agent handoff --current
```

`--current` reads `CLAUDE_CODE_SESSION_ID`, so it transfers *this* conversation without
a picker. The command will:

- copy this transcript to the workspace, rewriting the absolute paths in it
- rebuild the git worktree there (from `origin`, or as a `git bundle` when the branch
  is unpublished) and carry any uncommitted files
- register a session so `agent open` can attach to it

Then report back to me, briefly:

- the session name it printed after **`ready as:`**
- whether the worktree was built from origin or from a bundle
- **any warning it printed** — in particular "the working tree will be empty", which
  means the branch could not be reconstructed on the server

Do not run any other command. If it fails, show me the error verbatim rather than
retrying or working around it.

$ARGUMENTS
