---
name: 5dive-cli
description: Use the local `5dive` CLI on a 5dive runtime VM to spawn, inspect, send to, and tear down sibling agents. Trigger this skill whenever the user asks for a worker, sub-agent, side task, parallel run, "another agent", "fan out", "delegate", or anything that needs more than one Claude/Codex/Gemini process running at once on the host. Also trigger when the user asks to inspect, restart, or pair an existing agent, when they mention `/var/lib/5dive/`, or when they need a machine-readable health check (`5dive doctor --json`). Always prefer `5dive` over running coding CLIs by hand — it is the only sanctioned way to keep agents under systemd.
---

# 5dive-cli

This skill teaches you to drive the `5dive` command on a 5dive runtime VM.
You are running inside one such VM. You can spawn additional agents on the
same host by shelling out to `sudo 5dive ...` and parsing the JSON envelope
it emits when you pass `--json`.

## When to use this skill

Use it whenever the work in front of you would benefit from a second pair of
hands — for example:

- The user asks for a "worker", "sub-agent", "another agent", or "side task".
- A long task could fan out into independent pieces (e.g. audit each route
  in parallel, run a different model on the same prompt, A/B two implementations).
- You need to keep one agent on a hot context while a second one investigates
  something orthogonal.
- The user wants to inspect / restart / pair / tear down an agent that
  already exists on the host.
- You need a machine-readable health check of the host's coding-CLI stack.

If the user just wants you to do the work yourself, do not spawn an agent.

## Mental model

Everything the CLI does maps onto these resources on the host:

- One **agent** = one Linux user (`agent-<name>`) + one systemd unit
  (`5dive-agent@<name>.service`) + one tmux session (`agent-<name>`)
  running the chosen CLI in a restart loop.
- Auth is decoupled. You authenticate a *type* once; every agent of that
  type inherits the credentials via `EnvironmentFile`.
- A **channel** (`telegram` / `discord` / `none`) is the inbound message
  surface. Channels are only supported by `claude`, `openclaw`, `hermes`.
- The CLI is idempotent and safe to call from another agent — your agent
  user is in the `claude` group and has `sudo 5dive ...` whitelisted.

## Output contract — always pass `--json`

Pass `--json` as a global flag (anywhere on the command line). Stdout
becomes a stable envelope; progress lines stay on stderr.

```bash
sudo 5dive agent create scout --type=claude --json
```

Success:
```json
{ "ok": true, "data": { "name": "scout", "type": "claude", "created": true } }
```

Failure (exit code matches `error.code`):
```json
{ "ok": false, "error": { "code": 6, "class": "auth_required", "message": "..." } }
```

**Branch on `error.class`, not on the human message.** Classes:
`ok`, `usage`, `validation`, `not_found`, `conflict`, `auth_required`,
`not_installed`, `not_running`, `pairing`, `permission`, `timeout`, `generic`.

See `references/exit-codes.md` for the full table.

## Recipes

### Spawn a worker for a side task

```bash
# 1. Pick a unique name (lowercase letters/digits/hyphens, ≤16 chars,
#    must start with a letter). Check the registry first if you care:
sudo 5dive agent list --json | jq '.data.agents | keys'

# 2. Create the worker. --workdir scopes its tmux cwd; default is
#    /home/claude/projects.
sudo 5dive agent create worker-1 \
  --type=claude \
  --workdir=/home/claude/projects/myrepo \
  --json

# 3. Send it the task. tmux send-keys + Enter, so the text appears
#    in the worker's running CLI prompt.
sudo 5dive agent send worker-1 \
  "audit the auth middleware for OWASP A01 issues; report back as a markdown bullet list"

# 4. Poll its output until it goes idle. --tmux dumps the scrollback.
sudo 5dive agent logs worker-1 --tmux --lines=80

# 5. Tear it down when you're done — frees the systemd unit + Linux user.
sudo 5dive agent rm worker-1 --json
```

### Fan out: same prompt, three different models

Useful for "let me see how Codex/Gemini/Claude each approach this".

```bash
for type in claude codex gemini; do
  sudo 5dive agent create "fan-${type}" --type="${type}" --json
  sudo 5dive agent send "fan-${type}" "$PROMPT"
done

# Wait, then collect the last 200 lines of each:
for type in claude codex gemini; do
  echo "=== ${type} ==="
  sudo 5dive agent logs "fan-${type}" --tmux --lines=200
done

# Cleanup.
for type in claude codex gemini; do
  sudo 5dive agent rm "fan-${type}" --json
done
```

### Recover from `auth_required`

```bash
# If create fails with error.class=auth_required, the type isn't authenticated.
# Two paths — pick by what credentials you have:

# A) Static API key in $KEY (preferred for automation)
echo "$KEY" | sudo 5dive agent auth set claude --api-key=- --json

# B) Device-code flow (when only a human can complete login)
sudo 5dive agent auth start claude --json
# -> session id; give the URL from `auth poll` to the user; they paste the
#    callback code back via `auth submit`.
```

Never call `5dive agent auth login <type>` from your own process — it
hands the TTY off to the upstream CLI's interactive flow and hangs your
agent. Use `auth start` / `auth set` instead.

### Diagnose a sick host

```bash
sudo 5dive doctor --json
```

Envelope is always `{ ok: true, data: { summary, checks } }` with exit 0.
Branch on `data.summary.errors > 0`. Add `--repair` to attempt reversible
fixes (apt installs, type installer recipes, registry reseed).

## Rules of engagement

1. **Always pass `--json`.** Parse the envelope. Don't grep stderr.
2. **One name = one agent.** Names are lowercase letters/digits/hyphens,
   start with a letter, max 16 chars. Reuse a name only after `agent rm`.
3. **Don't share bot tokens.** Two Telegram-channel agents on the same
   bot will race each other on `getUpdates`. Each agent needs its own.
4. **Tear down what you spin up.** A leaked `worker-N` agent stays
   running across reboots — it's a real systemd unit, not a thread.
   On task completion call `5dive agent rm <name>`.
5. **Don't shell out to the underlying CLI binaries directly.** Going
   around `5dive` skips the systemd unit, the audit log, and the env
   injection — the agent will run with broken auth and no restart loop.
6. **Read `5dive --help`** if a flag is rejected as unknown — the binary
   on the host may be newer or older than this skill. The help output is
   authoritative.
7. **The `auth login <type>` path is interactive only.** Never call it
   from your own session.

## Reference

- `references/commands.md` — every subcommand and flag, copy/pasteable.
- `references/exit-codes.md` — exit codes & error classes.
- `references/paths.md` — on-disk state layout (only for debugging).

## Going further

The full reference manual lives at <https://5dive.com/docs>. If a flag in
this skill conflicts with what the running binary accepts, trust the
binary — run `sudo 5dive --help` or `sudo 5dive agent <sub> --help`
directly and follow that.
