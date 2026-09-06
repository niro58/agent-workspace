# Project secrets

Your repos clone cleanly onto the server, and their `.env.example` files come with
them — but the real `.env` files are gitignored, so the workspace gets a checkout that
can't actually run anything. `docker compose` starts, and then your app has no database
URL.

`agent env` closes that gap without putting secrets in any project repo.

## Shape

A separate **private** vault repo, one directory per project:

```
env-vault/
├── .sops.yaml
├── driver-hub/
│   ├── .env
│   └── .env.prod
└── my-app/
    ├── .env
    └── packages/api/.env
```

Files are encrypted with [sops](https://github.com/getsops/sops) in `dotenv` mode:
**values are encrypted, key names stay readable.** So `git diff` tells you *which*
variable changed without revealing it, and a review is meaningful.

A vault repo rather than per-project encrypted files, because the latter means a commit
and a pull request against every repo you own. The vault touches none of them.

## Where the key lives

The age private key stays on machines you trust, at `~/.config/sops/age/keys.txt`. It is
**deliberately not placed in the workspace pod.**

That matters: the pod runs a privileged Docker sidecar, and a workspace holding the key
could decrypt the vault for every project — including ones it has never been given.
Instead the workspace receives *already-decrypted* files. The cost is that you seed a
project once from a trusted machine; after that the files persist on the volume.

If you'd rather the server be autonomous, mount the key as a Kubernetes Secret and
run `agent env pull` inside the pod. Understand the trade before you do.

> **Back the key up.** It's one short text file, and losing it means losing every value
> in the vault. A password manager entry is fine.

## Setup

```bash
# 1. an age identity
mkdir -p ~/.config/sops/age
age-keygen -o ~/.config/sops/age/keys.txt
chmod 600 ~/.config/sops/age/keys.txt

# 2. a private vault repo
mkdir -p ~/.local/share/agent-env-vault && cd $_
cat > .sops.yaml <<EOF
creation_rules:
  - path_regex: .*
    age: $(age-keygen -y ~/.config/sops/age/keys.txt)
EOF
git init -b main && git add -A && git commit -m "Initialise vault"
gh repo create <you>/env-vault --private --source=. --remote=origin --push
```

Point `agent` at it if you used a different path:

```bash
echo 'AGENT_VAULT=$HOME/.local/share/agent-env-vault' >> ~/.config/agent-workspace/config
```

## Use

```bash
cd ~/projects/my-app
agent env status    # what's on disk, in the vault, and in the workspace
agent env push      # encrypt this repo's gitignored .env files into the vault
agent env ship      # decrypt and copy them into the workspace's clone
agent env pull      # decrypt them back onto this machine (e.g. a new laptop)
```

Shipped files are written `chmod 600`.

## What gets collected

Files matching `.env*` that git **ignores**, excluding:

- `*.example` — tracked in the repo, so already present after a clone
- `*.bak`, `*.bak-*`, `*.bak.*` — timestamped backups; you don't want these versioned
- anything under `node_modules/`, `.git/`, or a worktree directory
  (`.claude/worktrees/`, `.worktrees/`) — worktree copies are derived state

Run `agent env status` first; it prints exactly what would be taken.

## Two things that will bite you

**Don't put `.env` in the vault's `.gitignore`.** It's the reflex from every other repo,
and here it silently drops every file literally named `.env` while the `.env.prod`
variants sail through. You get a vault that looks populated and isn't. Only `keys.txt`
and `*.age` belong in that ignore file.

**sops resolves `.sops.yaml` relative to the file it's encrypting**, not the working
directory. Encrypting a file inside a project directory will fail with *"config file not
found, or has no creation rules"* unless you pass `--config` explicitly. `agent env`
does this for you.

## Round-trip fidelity

sops's dotenv handler drops blank lines, so a decrypted file may have fewer lines than
the original. **Key/value pairs are byte-identical**; only cosmetic spacing is lost,
which no `.env` parser cares about. If you need byte-exact files, encrypt them as binary
instead — at the cost of readable diffs.
