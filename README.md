# claude-skills

Claude Code skills. Currently Salesforce/Flexport-flavoured.

## Skills

### `jira-estimate`

Pre-reads Jira tickets before grooming. Fetches from Jira, flags unclear points, estimates story points (0.5–8), logs actuals for calibration over time.

```
/jira-estimate WOLF-3842 WOLF-3830 WOLF-3833
```

**Prerequisites:** Claude Code + Atlassian MCP configured.

## Install

```bash
git clone https://github.com/gmendesmoreira/claude-skills /tmp/claude-skills
cp -r /tmp/claude-skills/jira-estimate ~/.claude/skills/
```

Restart Claude Code. Done.

## Contributing

PRs welcome.
