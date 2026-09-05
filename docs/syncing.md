# Syncing config between your machine and the workspace

This is the only bidirectional layer in the system. Code travels by git; sessions don't
travel at all. Config is the one thing that genuinely has to exist, identically, in two
places.

## Why not just copy `~/.claude`

Because most of it isn't config. A working `~/.claude` is typically several gigabytes,
almost entirely conversation transcripts, file history and caches. The part that actually
defines how your agent behaves is on the order of a couple hundred kilobytes.

Copy the whole directory and you're syncing gigabytes of churn, plus credentials you
should never move between machines.

## The split

**Track:**

```
.claude/CLAUDE.md              your global instructions
.claude/settings.json          model, hooks, permissions, plugins
.claude/hooks/                 the scripts those hooks call
.claude/agents/                subagent definitions
.claude/skills/                skills you wrote or installed
.claude/statusline-script.sh
.codex/config.toml
```

**Never track:**

```
.claude/.credentials.json      log in per machine instead
.codex/auth.json
.claude/projects/              transcripts — this is the multi-GB part
.claude/file-history/
.claude/history.jsonl
.claude/cache/  backups/  sessions/  shell-snapshots/  paste-cache/
.claude/plugins/               reinstall from the marketplace instead
**/node_modules
```

## Host-specific values

A few things genuinely differ per machine, and a blind copy breaks them:

| | Desktop | Server |
|---|---|---|
| Hook and statusline paths | your home | the container's home |
| MCP servers pointing at localhost | present | usually absent |
| Project trust roots | `~/projects` | `~/work` |
| Desktop-only skills | present | excluded |

[chezmoi](https://chezmoi.io) handles this with templates. Set a `profile` variable per
machine:

```toml
# ~/.config/chezmoi/chezmoi.toml
[data]
    profile = "desktop"     # or "server"
```

Then template the values that differ:

```
# settings.json.tmpl
"command": "bash {{ .chezmoi.homeDir }}/.claude/hooks/block-push-to-main.sh"
```

```
# config.toml.tmpl
{{ if eq .profile "desktop" -}}
[mcp_servers.something]
command = "{{ .chezmoi.homeDir }}/.local/bin/something"
{{- else -}}
[projects."{{ .chezmoi.homeDir }}/work"]
trust_level = "trusted"
{{- end }}
```

And exclude by profile in `.chezmoiignore`:

```
{{- if ne .profile "desktop" }}
.claude/skills/my-desktop-only-skill
{{- end }}
```

## Verifying before you commit

The point of templating is that applying the repo on your desktop changes **nothing**.
Check that before you push:

```bash
chezmoi diff      # should be empty
chezmoi status    # should be empty
```

If it isn't, your template renders differently from the file it replaced, and you'd be
about to modify your working setup.

## Day to day

```bash
agent sync
```

Re-adds tracked files, commits, pushes, then pulls and applies in the pod.

## Binary dependencies your hooks rely on

If a hook shells out to a tool, that tool must exist in the pod or **every command
fails**. This is easy to miss because the failure is confusing rather than obvious.

Two gotchas:

- A binary built on a rolling-release distro often needs a newer glibc than a stable
  container base provides. Prefer a musl/static build.
- Pin it to the version on your desktop, so hook behaviour matches.

Put the install in the manifest's `install-tools.sh`, not in a one-off `kubectl exec` —
otherwise a rebuilt pod comes up without it.
