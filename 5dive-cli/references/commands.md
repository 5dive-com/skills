# 5dive CLI — full command reference

Every subcommand accepts `--json` as a global flag. The exact help output
on the host is authoritative; this file is the canonical reference shape.

## Top-level

```
5dive agent ...                      # agent CRUD
5dive doctor [--repair] [--category=deps|types|auth|registry|shelld]
5dive --help | -h | help
```

## Agents

```
5dive agent list
5dive agent types
5dive agent create <name> --type=<type>
                          [--channels=none|telegram|discord]
                          [--telegram-token=<bot-token>]
                          [--discord-token=<token>]
                          [--workdir=<path>]
                          [--auth-profile=<name>]
5dive agent clone <src> <dst> [--channels=...] [--telegram-token=...]
                              [--discord-token=...] [--workdir=...]
5dive agent start   <name>
5dive agent stop    <name>
5dive agent restart <name>
5dive agent rm      <name>
5dive agent stats   <name>
5dive agent logs    <name> [--follow] [--lines=N] [--tmux]
5dive agent send    <name> <text...>
5dive agent <name>  tui                       # attach this terminal
5dive agent install <type>                    # install the type's binary
```

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
5dive agent pair <name>                        # returns a code; user DMs the bot
5dive agent pair <name> --code=<code>          # paste bot reply or bare code
```

## Auth

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

## Skills

```
5dive agent skill <name> add  --source=<owner/repo> --skill=<id>
5dive agent skill <name> list
5dive agent skill <name> rm   <skill-id>
```

## Doctor

```
5dive doctor                                   # text
5dive doctor --json                            # always exit 0; branch on data.summary.errors
5dive doctor --repair                          # attempt reversible fixes
5dive doctor --category=deps                   # also: types | auth | registry | shelld
```

## Known agent types (default install)

| Type      | Channels | Notes |
| --------- | -------- | ----- |
| claude    | yes      | Anthropic Claude Code |
| codex     | no       | OpenAI Codex CLI |
| gemini    | no       | Google Gemini CLI |
| hermes    | yes      | Nous Research hermes harness (OpenAI-backed) |
| openclaw  | yes      | Third-party Claude harness (OpenAI-backed) |
| opencode  | no       | opencode.ai (free models, no signup) |
