# agent-workspace

Run your terminal AI agents (Claude Code, Codex) on a server instead of your laptop, so
they keep working when your machine is off — and hand a running conversation over
mid-thought.

Your laptop becomes a viewport. The work lives on the server.

```bash
cd ~/projects/my-app
agent                 # opens a persistent session on the server, in that repo
# close the lid, go home, open a terminal
agent                 # you're back in the same session, still running
```

---

## Why

If you work primarily in a terminal with an AI agent, the session *is* the work. But a
terminal session dies with your laptop: you close the lid, the SSH drops, the agent's
half-finished train of thought is gone.

The obvious fix — "sync my files to a server" — is the wrong primitive. Bidirectional
file sync plus an agent rewriting files plus git metadata equals constant conflicts.
`.git/index`, lockfiles, and `node_modules` will fight you, and a worktree-heavy workflow
(one checkout per parallel agent) generates churn no sync tool survives.

The primitive that works: **there is one copy, on the server, and you attach to it.**
Nothing to reconcile, because nothing is duplicated.

## What you get

- **Persistent sessions.** One local terminal maps to one long-lived remote session.
  Detaching, closing the terminal, or shutting down the machine all leave it running.
- **A full dev environment.** Docker and `docker compose`, so projects that expect
  `postgres` + `redis` + a built image just work. Playwright with a real browser.
  node, pnpm, git, `gh`, kubectl.
- **Config parity.** Your agent instructions, subagents, and skills follow you to the
  server via a dotfiles repo, with host-specific values templated.
- **Conversation handoff.** `agent handoff` ships the Claude Code conversation you're
  in right now to the server and reopens it there, already resumed. You type `continue`.

## Requirements

| | |
|---|---|
| A Kubernetes cluster | k3s on a single VPS is plenty. Tested on k3s v1.32. |
| `kubectl` access to it | The workspace never needs SSH — useful if port 22 is firewalled. |
| A `local-path` (or any RWO) StorageClass | Two PVCs: 50Gi for `$HOME`, 20Gi for the Docker image cache. |
| Node headroom | ~2 CPU / 4 GB free is workable; more is better if you run parallel agents. |
| Locally: `kubectl`, `git`, `bash` | `fzf` optional but makes the session pickers nicer. |
| A Claude and/or OpenAI subscription | You log in inside the pod once. |

Nothing needs to be installed on the *node* itself.

## Install

```bash
git clone https://github.com/<you>/agent-workspace
cd agent-workspace

# 1. create the workspace
kubectl apply -f k8s/workspace.yaml

# 2. put the launcher on your PATH
install -m 755 bin/agent ~/.local/bin/agent

# 3. point it at your cluster (optional if you only have one context)
mkdir -p ~/.config/agent-workspace
cat > ~/.config/agent-workspace/config <<'EOF'
CTX=my-cluster        # kubectl context; omit to use the current one
NS=agent              # namespace
EOF
```

Then, once the pod is running:

```bash
agent gh-login        # log the workspace into GitHub
agent                 # opens a session; inside it run `claude` and `codex` to log in
```

First boot takes a couple of minutes — an init container installs the browser
system libraries, and the workspace downloads its toolchain onto the volume.
Subsequent restarts are seconds, because everything persists.

## Commands

| | |
|---|---|
| `agent` | Resume a free session for this checkout, else open the next one |
| `agent -n [label]` | Force a new session — `-n api` gives `repo@api` |
| `agent -m` | Deliberately mirror a session already in use |
| `agent -l` | This project's sessions, with client counts |
| `agent resume` | Picker over every session, current project first |
| `agent ls` | Everything running |
| `agent wt` | Open a session in a server-side git worktree |
| `agent handoff` | Move this checkout's Claude conversation to the server, resumed |
| `agent fetch` | Bring a conversation back from the server to this machine |
| `agent fix` | Repair a corrupted or wrong-sized terminal |
| `agent killall` | Kill every session for this project |
| `agent env <cmd>` | Encrypted project secrets — `push` / `pull` / `ship` / `pull-remote` / `status` |
| `agent sync` | Two-way config sync between this machine and the workspace |
| `agent shell` | Plain bash, no multiplexer |

Sessions are named `repo`, `repo@2`, `repo@label`. Bare `agent` never mirrors: it
reuses a session nobody is attached to, and only creates a new one when they're all
busy. That single rule covers both "give me my session back" and "give me another
terminal", which otherwise collide.

## How the syncing works

Three layers, and only one is bidirectional. That's deliberate — it's why there is no
conflict resolution anywhere in the system.

**Code — git, cloned on demand.** The server clones a repo into `$WORKDIR/<repo>` the
first time you run `agent` in it. GitHub is the sync layer; it always was. Nothing is
mirrored, so the size of your local projects directory is irrelevant.

**Worktrees — recreated, never synced.** A worktree is derived state, fully
reconstructible from `git worktree add`. Branches travel between machines; directories
don't. This is what makes a worktree-heavy agent workflow viable at all.

**Config — genuinely two-way, via chezmoi.** `agent sync` checks the *pod first* and
commits anything changed there before applying local changes. A one-way sync would
silently discard config you edited on the server.

**Config — bidirectional, via chezmoi.** Your agent instructions, subagents, skills and
hooks live in a git repo applied on both machines. Credentials are excluded — you log in
once per machine. Host-specific values (absolute paths, MCP servers pointing at
localhost, desktop-only skills) are templated on a `profile` variable, so one repo serves
both.

See [docs/syncing.md](docs/syncing.md) for the chezmoi layout.

**Project secrets — a separate encrypted vault.** Your `.env` files are gitignored, so a
clean clone on the server can't actually run anything. `agent env` keeps them in a
private sops+age vault, one directory per project, and ships decrypted copies into the
workspace. The age key stays on your machine and never enters the pod.
See [docs/secrets.md](docs/secrets.md).

**Sessions — single-homed on purpose.** They exist only on the server.

> **The one discipline this demands:** uncommitted work does not travel. Commit and push
> before switching machines, or it is stranded on whichever box it's on. That is the
> price of having no merge conflicts anywhere else.

## Architecture

```
workspace pod
├── init 1  browser-deps   installs Playwright's system libraries (root), once
├── init 2  fix-perms      hands the volume to uid 1000
├── workspace              node:22-bookworm, uid 1000 — your terminals
│                          DOCKER_HOST=unix:///var/run/docker-shared/docker.sock
├── dind                   docker:27-dind, privileged — the Docker daemon
└── volumes                home 50Gi · docker 20Gi · shared socket (emptyDir)
```

Both containers share a network namespace, which is why `docker compose` port
mappings land on the workspace's own `localhost` — exactly as they would on your laptop.

The volume mounts at `/home/node` rather than a custom path because OpenSSH resolves
`~` from the passwd entry, not `$HOME`, and uid 1000 in that image is `node`. Mounting
where the container's user actually lives avoids special-casing git, ssh and everything
downstream.

The workspace runs as uid 1000, not root. This is required: Claude Code refuses
`--dangerously-skip-permissions` when running as root.

## Security

**The `dind` sidecar is `privileged: true`.** That container can reach the node. This is
the cost of real `docker compose`, and you should decide whether it's acceptable on your
node before deploying — especially if the node runs anything else.

What this setup *avoids*: the standard Docker-in-Docker recipe exposes
`tcp://0.0.0.0:2375`, an unauthenticated root-equivalent API reachable by every pod in
the cluster. Here `dockerd-entrypoint.sh` is bypassed so there is **no network listener
at all**. The daemon socket lives on a shared `emptyDir`, group-owned by gid 1000:

```
srw-rw---- 1 root node 0 docker.sock
```

If you don't need Docker, delete the `dind` container, the `agent-docker` PVC and the
`dockersock` volume from the manifest. Everything else works without it.

## Conversation handoff

Claude Code stores transcripts at
`~/.claude/projects/<cwd with / and . replaced by ->/<session-uuid>.jsonl`, and
`claude --resume <uuid>` reopens one.

`agent handoff` picks a conversation, rewrites the absolute paths inside it from your
local checkout to the server's, copies it into the pod, and opens a session booted
straight into `claude --resume`. The conversation arrives with its full context.

Caveats worth knowing:

- The transcript format is internal to Claude Code and may change between versions.
- Path rewriting covers the repo path. References to files outside it stay as-is —
  harmless, but the agent may mention paths that don't exist on the server.
- Don't resume the same conversation on two machines at once. Use `--fork-session`
  (Claude Code's own flag) if you want a branch rather than a move.
- Large transcripts are large. The command warns above 60 MB.

## Troubleshooting

See [docs/troubleshooting.md](docs/troubleshooting.md). The two you're most likely to
hit:

- **The UI is stuck at the wrong size.** A dead client is pinning the session — zellij
  sizes a session to its *smallest* attached client. Run `agent fix`.
- **`playwright` fails with exit 127.** The browser system libraries aren't present.
  That's what the `browser-deps` init container installs; check its logs.

## Limitations

- Single replica. The PVC is RWO and sessions are single-homed by design.
- If the pod restarts, running processes die. Files and transcripts survive on the
  volume; `claude --resume` gets the conversation back.
- Browser-driven tooling that needs your actual desktop browser can't move to the
  server. Playwright can; a browser-extension integration can't.
- Most tool versions float — they resolve to latest at build time. Pin them in
  `install-tools.sh` if you need reproducibility.

## License

MIT
