# SkillSafe plugin

Scan agent skills for security issues before you install them, and check your
own skills before you publish them, with the free
[SkillSafe](https://skillsafe.ai) scanner. Works in Codex and Gemini CLI. No
account or API key is needed.

Third-party skills can carry prompt injection, credential theft, data
exfiltration, or hidden persistence. SkillSafe checks for these patterns and
reports what a skill can actually do: the network destinations it contacts,
the environment variables it reads, the commands it runs, and the files it
touches.

## Skills

| Skill | What it does |
|---|---|
| `scan-before-install` | Given a GitHub URL, scans the skill or plugin there and reports a grade (A+ to F), the number of findings, and a link to the full report. It only scans; installing stays your decision. |
| `check-skill-before-publishing` | Checks a skill you wrote, explains each finding, helps you fix it, and points you to where you can publish it. |

Try asking:

- "Is this skill safe to install? https://github.com/OWNER/REPO"
- "Check my skill in ./my-skill before I publish it."

## Install

**Codex**

```bash
codex plugin marketplace add skillsafe/skillsafe-plugin
codex plugin add skillsafe
```

**Gemini CLI**

```bash
gemini extensions install https://github.com/skillsafe/skillsafe-plugin
```

## What the plugin sends, and where

The plugin contains instructions only: no scripts, hooks, or MCP servers. When
a skill runs, the agent makes these requests, all to `api.skillsafe.ai`:

| Skill | Request | What is sent |
|---|---|---|
| `scan-before-install` | `POST https://api.skillsafe.ai/v1/scan/github` | The public GitHub repository URL and, optionally, a folder path. SkillSafe fetches the public repository itself; nothing from your machine is uploaded. Scan results are stored and shown on skillsafe.ai. |
| `check-skill-before-publishing` | `POST https://api.skillsafe.ai/v1/scan/files` | The text contents of the files in your skill folder, sent only after you confirm. Credential files such as `.env` are excluded. The files are analyzed and returned a result; they are not stored. |

Nothing is published to SkillSafe unless you do it yourself afterwards, from
the [desktop app](https://app.skillsafe.ai/) or the
[web dashboard](https://skillsafe.ai/dashboard/).

See the [privacy policy](https://skillsafe.ai/privacy-policy/) and the
[published detection rules](https://skillsafe.ai/security/).

## Limits

A clean grade means no known risky patterns were found, not that a skill is
guaranteed safe. Scans are point-in-time; re-scan when a repository changes.

## License

MIT
