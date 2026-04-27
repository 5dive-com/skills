# 5dive skills

Skills published by [5dive](https://5dive.com) for use with
[skills.sh](https://skills.sh/) and any agent harness that loads
`SKILL.md` bundles (Claude Code, openclaw, hermes, …).

| Skill                  | Purpose                                                       |
| ---------------------- | ------------------------------------------------------------- |
| [`5dive-cli`](./5dive-cli) | Drive the local `5dive` CLI on a 5dive runtime VM. Lets an agent spawn, inspect, send to, and tear down sibling agents on the same host. |

## Install on a 5dive agent

From the dashboard: **Agents → Connect skills → 5dive-cli → Install**.

From the host:

```bash
sudo 5dive agent skill <agent-name> add \
  --source=5dive-com/skills \
  --skill=5dive-cli
```

Or via the upstream `skills` CLI from any agent's home:

```bash
npx skills add https://github.com/5dive-com/skills --skill 5dive-cli --agent claude-code --yes
```

## License

MIT.
