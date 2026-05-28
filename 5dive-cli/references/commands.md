# 5dive CLI — full command reference

Every subcommand accepts `--json` as a global flag. The exact help output
on the host is authoritative; this file is the canonical reference shape.

## Top-level

```
5dive agent   ...                    # agent CRUD
5dive account ...                    # named auth profiles
5dive task    ...                    # host-shared task queue (no sudo)
5dive org     ...                    # agent org chart (no sudo)
5dive doctor  [--repair] [--category=deps|types|auth|registry|shelld]
5dive --help | -h | help
```

## Agents

```
5dive agent list
5dive agent types
5dive agent create <name> --type=<type>
                          [--channels=none|telegram|discord]
                          [--telegram-token=<bot-token>]
                          [--telegram-home-channel=<chat-id>]   # hermes only
                          [--telegram-allowed-users=<csv-ids>]  # hermes only
                          [--discord-token=<token>]
                          [--workdir=<path>]
                          [--auth-profile=<name>]
                          [--isolation=admin|standard|sandboxed] # default admin
                          [--with-skills=<spec>[,<spec>...]]    # bare id (5dive-com/skills) or owner/repo:id
                          [--no-skills]                          # opt out (overrides agent-spawn default)
                          [--defer-auth]                         # skip auth gate; first-run UI handles it
5dive agent clone <src> <dst> [--channels=...] [--telegram-token=...]
                              [--discord-token=...] [--workdir=...]
5dive agent start   <name>
5dive agent stop    <name>
5dive agent restart <name>
5dive agent rm      <name>
5dive agent stats   <name>
5dive agent logs    <name> [--follow] [--lines=N] [--tmux]
5dive agent send    <name> <text...> [--from=<sender>] [--raw]
5dive agent ask     <name> <text...> [--from=<sender>] [--timeout=120] [--idle-secs=5] [--poll-secs=2]
5dive agent <name>  tui                       # attach this terminal
5dive agent install <type>                    # install the type's binary
5dive agent set-account <agent> <account|default>   # alias for `agent config set auth-profile=`
```

When `agent create` runs from another agent (`SUDO_USER=agent-*`),
`--with-skills=5dive-cli` is the default for every supported type. Pass
`--no-skills` to opt out, or `--with-skills=...` to override the list
explicitly.

## Config

```
5dive agent config <name> set channels=<none|telegram|discord>
5dive agent config <name> set workdir=<path>          # "default" clears
5dive agent config <name> set auth-profile=<name>     # "default" clears
5dive agent config <name> set telegram.token=<token>
5dive agent config <name> set discord.token=<token>
```

## Pairing (Telegram / Discord)

```
5dive agent pair <name>                                       # returns a code; user DMs the bot
5dive agent pair <name> --code=<code>                         # paste bot reply or bare code
5dive agent pair <name> --user-id=<id> [--chat-id=<id>]       # seed access.json directly; chat_id defaults to user_id (private DM)
5dive agent telegram-discover --token=<bot-token> [--poll-secs=N]    # long-poll getUpdates; returns {found, userId, chatId, ...}
5dive agent telegram-getme    --token=<bot-token>                    # getMe — {botId, username, firstName}
```

## Auth (lower-level — prefer `account` for human flows)

```
5dive agent auth status [--probe] [--type=<type>]
5dive agent auth login  <type>                 # TTY only — never from a script
5dive agent auth set    <type> --api-key=<key|->
                        [--auth-profile=<name>]
5dive agent auth start  <type> [--auth-profile=<name>]
5dive agent auth poll   <session_id>
5dive agent auth submit <session_id> --code=<callback>
5dive agent auth cancel <session_id>
```

## Accounts (named auth profiles)

```
5dive account list                                   # name, types signed in, # agents bound
5dive account show   <name>                          # detail incl. envKeys present
5dive account add    <name>                          # create empty account; sign in next
5dive account login  <name> --type=<type>            # TTY-only interactive login into an account
5dive account rename <old> <new>                     # repoints + restarts every bound agent
5dive account remove <name>                          # refuses if any agents still bound
```

The reserved name `default` is rejected by `account add` / `rename`.

## Skills

```
5dive agent skill <name> add  --source=<owner/repo> --skill=<id>
5dive agent skill <name> list
5dive agent skill <name> rm   <skill-id>
```

## Tasks (shared queue)

Host-shared task queue at `/var/lib/5dive/tasks/tasks.db` (sqlite, WAL). The
store is group-writable, so every `agent-*` user can read/write **without
sudo**. Tasks get a `DIVE-N` ident; statuses are
`todo | in_progress | blocked | done | cancelled`.

```
5dive task add <title...> [--body=<text>] [--priority=low|medium|high|urgent]
                          [--assignee=<agent>] [--parent=<id|DIVE-N>] [--from=<who>]
5dive task ls   [--status=<s>] [--assignee=<agent>] [--mine] [--all]  # default: open, priority-ordered
5dive task show <id|DIVE-N>                       # detail + subtasks + blockers
5dive task assign <id|DIVE-N> <agent>
5dive task start  <id|DIVE-N>                     # -> in_progress
5dive task done   <id|DIVE-N>                     # -> done
5dive task cancel <id|DIVE-N>                     # -> cancelled
5dive task block   <id|DIVE-N> --by=<id|DIVE-N>   # add a blocks edge, mark blocked
5dive task unblock <id|DIVE-N> [--by=<id|DIVE-N>] # drop edge(s); back to todo if clear
5dive task rm <id|DIVE-N>                         # delete (cascades subtasks + edges)
```

`task init` is a one-time root bootstrap run at provision time — agents never
call it. From an agent, `--from` defaults to your `agent-*` name (or `SUDO_USER`
when run via sudo), so `created_by` is attributed automatically.

## Org chart

Who-reports-to-whom across agents, in the same group-writable store (no sudo).
A single self-referential `reports_to` edge per agent, plus optional role/title.

```
5dive org set <agent> [--manager=<agent>|default] [--role=<text>] [--title=<text>]  # upsert; --manager=default clears
5dive org tree                                    # whole hierarchy, indented
5dive org show <agent>                             # manager + direct reports
5dive org ls                                       # flat list of everyone placed
5dive org rm <agent>                               # remove (reports re-parent to null)
```

## Doctor

```
5dive doctor                                   # text
5dive doctor --json                            # always exit 0; branch on data.summary.errors
5dive doctor --repair                          # attempt reversible fixes
5dive doctor --category=deps                   # also: types | auth | registry | shelld
```

## Known agent types (default install)

| Type        | Channels | Notes |
| ----------- | -------- | ----- |
| antigravity | no       | Google Antigravity CLI (binary: `agy`) |
| claude      | yes      | Anthropic Claude Code |
| codex       | yes      | OpenAI Codex CLI |
| gemini      | no       | Google Gemini CLI |
| grok        | no       | xAI Grok CLI |
| hermes      | yes      | Nous Research hermes harness (OpenAI-backed) |
| openclaw    | yes      | Third-party Claude harness (OpenAI-backed) |
| opencode    | no       | opencode.ai (free models, no signup) |

`channels=yes` means the type can take `--channels=telegram` (and, for
claude/openclaw, `--channels=discord`). Run `5dive agent types` on the host
for the authoritative list — installers add or drop entries over time.
