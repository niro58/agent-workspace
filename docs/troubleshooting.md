# Troubleshooting

Every entry here is something that actually happened during the build, with the
measurement that identified it.

## The terminal UI is stuck at the wrong size

**Symptom.** Content renders in a small box in the corner while the multiplexer frame
draws at full size. Resizing the window doesn't help.

**Cause.** zellij sizes a session to its **smallest attached client**. A `kubectl exec`
that died without clean teardown leaves a client process behind holding a stale pty. One
real case:

| client | pty size |
|---|---|
| the actual terminal | 61 × 270 |
| stale | 11 × 47 |
| stale | 11 × 19 |
| stale | 12 × 19 |

The 11×19 client pinned the whole session to a corner.

**Fix.** `agent fix`. It SIGKILLs every client process for the session — the server
process and your panes survive — then reattaches at real geometry.

Note that `zellij delete-session` does **not** clear these: the client processes outlive
the session record, and a session can reappear after you "kill" it. Also, clients ignore
SIGTERM; it takes SIGKILL.

**Avoid causing it.** Never create sessions with `setsid script -qc "zellij attach"`.
`script` allocates a tiny pty and the client never exits. That is where the stale clients
in the table above came from.

## Playwright fails with exit 127

**Symptom.** Browsers download fine, then exit 127 with no useful message. No screenshot
is produced.

**Cause.** Missing system libraries — `libnss3`, `libatk-1.0`, `libgbm`, `libasound`,
`libxkbcommon`, `libcups`. `npx playwright install` fetches browsers but not these, and
`playwright install-deps` needs root, which the workspace deliberately doesn't have.

**Fix.** The `browser-deps` init container handles it: it runs as root, calls
`playwright install-deps`, diffs `/usr/lib/x86_64-linux-gnu` before and after, and copies
only the new libraries onto the volume. The workspace picks them up via
`LD_LIBRARY_PATH`. Because the init container uses the *same base image*, every copied
library is byte-identical in version, so nothing is shadowed.

If it's not working, check `kubectl logs <pod> -c browser-deps`. It's idempotent and
marked with a `.done` file, so to force a reinstall delete
`$HOME/.local/browserdeps/.done` and restart the pod.

**Don't** debug this with `ldconfig -p | grep`. The cache can be misleading; use `find`.

## Docker daemon won't start

**Symptom.** `Cannot connect to the Docker daemon`, and the dind logs show
`failed to load listeners: listen tcp 127.0.0.1:2375: bind: address already in use`.

**Cause.** `dockerd-entrypoint.sh` force-adds its own `--host=tcp://0.0.0.0:2375`. Any
`--host` you pass is *appended*, so specifying loopback collides with the entrypoint's
wildcard bind.

**Fix.** Bypass the entrypoint entirely — `command: ["dockerd"]` with explicit args. The
manifest does this, and it also removes the TCP listener, which you want anyway: the
default exposes an unauthenticated root-equivalent API to the whole cluster.

## Sessions mirror instead of multiplying

**Symptom.** Running `agent` twice shows the same content in both terminals.

**Cause.** zellij happily attaches N clients to one session. If the launcher computes one
session name per project, the second launch attaches a second *client*, not a second
workspace. Confirm with:

```bash
zellij --session <name> action list-clients
```

Three rows all showing `terminal_0` means three mirrors of one pane.

**Fix.** Handled by the naming scheme — `repo`, `repo@2`, … and the free/busy rule. Use
`agent -m` when you actually want to mirror.

## Sessions named after a worktree instead of the repo

**Cause.** `git rev-parse --show-toplevel` returns the *worktree* root, not the repo
root. Launching from a linked worktree names the session after the worktree slug and
clones the repo again under that name.

**Fix.** Use `--git-common-dir`, which resolves to the main checkout from both a worktree
and the main checkout itself:

```bash
dirname "$(git rev-parse --path-format=absolute --git-common-dir)"
```

Needs git ≥ 2.31.

## SSH inside the pod can't find its keys

**Cause.** OpenSSH resolves `~` from the **passwd entry**, not `$HOME`. If you set
`HOME=/home/agent` but uid 1000 is `node` with home `/home/node`, ssh reads
`/home/node/.ssh` and finds nothing.

**Fix.** Mount the volume where the container's user actually lives. The manifest uses
`/home/node` for exactly this reason.

## `kubectl cp` is slow or fails on a big transcript

Conversation transcripts can reach hundreds of megabytes. `agent handoff` warns above
60 MB. If a copy fails partway, delete the partial file in the pod and retry — a
truncated JSONL will not resume.

## Disk fills up

`local-path` enforces no quota, so a PVC's stated size is advisory; you're really
consuming the node's filesystem. Watch it, and run `docker system prune` inside the
workspace when the image cache grows.
