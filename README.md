# LeanLaw agent skills

Agent skills for law firms that use [LeanLaw](https://leanlaw.co). Each skill teaches an AI agent
(Claude, GitHub Copilot, or any agent that supports the open skills format) one piece of your firm's
billing workflow and does the work through your LeanLaw account.

These skills don't include a LeanLaw login of their own. They need the **LeanLaw MCP connector**,
which gives an agent access to your firm's clients, matters, time and invoices using the permissions
you approve. See [platform.leanlaw.co/agents](https://platform.leanlaw.co/agents) to set it up.

## Skills

| Skill | What it does | Status |
|---|---|---|
| `onboard-client` | Takes a new client from a signed engagement letter to a billable matter: runs a conflict pre-check, creates the client and matter, and sets up the fee arrangement, including fixed-fee installment schedules | In development |
| `weekly-attorney-dashboard` | Gives each attorney a weekly summary: hours by client and matter, progress toward a billable target, unbilled work, and invoices waiting on their review | In development |
| `missing-time-review` | Compares your calendar and sent email with the time already logged, finds work you haven't recorded, and drafts the missing entries for you to review | In development |

## How they work with your data

- **Nothing is written without your confirmation.** Before a skill creates a client, a matter, a
  fixed fee or a time entry, it shows you exactly what it will create and waits for your yes.
- **The connector's permissions still apply.** A skill can only do what you allowed when you
  authorized the connector, and the connector acts as you, with your role in LeanLaw.
- **Skills say what they can't do.** When a task needs something the connector doesn't offer, such
  as rate groups or trust deposits, the skill tells you to do it in
  [LeanLaw](https://myleanlaw.co) instead of guessing.

## Installing

First, connect the LeanLaw MCP connector to your agent. Instructions are at
[platform.leanlaw.co/agents](https://platform.leanlaw.co/agents).

### Claude Code

```
/plugin marketplace add leanlaw/agent-skills
/plugin install leanlaw@leanlaw-agent-skills
```

### Claude (claude.ai and the desktop app)

Download a skill's folder from `plugins/leanlaw/skills/`, zip it, and upload it under
**Settings → Capabilities → Skills**.

### Other agents

Each skill is a folder with a `SKILL.md` file that follows the open agent skills format. Copy the
folder to wherever your agent loads skills from, for example `.github/skills/` for GitHub Copilot.

## Support

Questions or feedback: contact LeanLaw support through [leanlaw.co](https://leanlaw.co).
