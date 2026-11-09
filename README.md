# agent-skills

Skills I use in Claude Code and Codex.

| Skill | What it does |
|---|---|
| `review-verify` | Reviews a change in a fresh agent, then checks each finding in a second one. |
| `review-verify-uncommitted` | The same for uncommitted work, without relying on session history. Needs `review-verify` installed. |

## Install

```bash
git clone git@github.com:IgnacioFDM/agent-skills.git
cp -R agent-skills/skills/* ~/.claude/skills/   # Claude Code
cp -R agent-skills/skills/* ~/.codex/skills/    # Codex
```
