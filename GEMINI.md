# SkillSafe extension

SkillSafe backs up the user's agent skills privately, turns them into hosted
web apps, and checks skills and MCP tools for security issues.

## Skills

Use these when the request matches; each one has full instructions.

- `skillsafe-sign-in`: connect the user's SkillSafe account (browser sign-in,
  or a temporary key to deploy right away). Run before the other two when no
  key is stored in `~/.skillsafe/config.json`.
- `back-up-skills`: back up the user's skills privately with version history,
  list backups, restore them on any machine, or delete a backup.
- `deploy-app`: turn the user's skills or an idea into a web app at
  `https://SLUG.skillsafe.ai`, private or shared.

## SkillSafe connector (MCP, no sign-in)

- `search_skills`, `recommend_skills`: find skills in the SkillSafe registry.
- `get_skill_info`, `get_install_commands`: details and install commands for one skill.
- `scan_skill`, `scan_mcp_config`: check skill files or MCP tool definitions for
  prompt injection, credential theft, and exfiltration.

Do not call SkillSafe for requests unrelated to agent skills, skill backups,
SkillSafe apps, or MCP tools. Never print or share the user's SkillSafe API key.
