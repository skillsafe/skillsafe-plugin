---
name: scan-before-install
description: Scan a GitHub-hosted agent skill or plugin for security issues with the free SkillSafe scanner before installing it. Use when the user asks to install a skill or plugin from GitHub, asks whether one is safe, or wants to vet third-party agent skills.
---

# Scan a skill before installing it

Before installing a third-party skill or plugin from GitHub, scan it with
SkillSafe's free public scanner and show the user the result. No account or API
key is needed.

**What this sends:** one request to `https://api.skillsafe.ai` containing the
public GitHub URL (and optional folder path) to scan. SkillSafe fetches the
public repository itself. Nothing from the user's machine is uploaded.

## Steps

1. **Work out the repository URL and folder.** For
   `https://github.com/OWNER/REPO/tree/main/skills/pdf`, the URL is
   `https://github.com/OWNER/REPO` and the subpath is `skills/pdf`. If the skill
   is at the repository root, there is no subpath.

2. **Run the scan.** The request returns when the scan has finished, usually
   within a minute:

   ```bash
   curl -s -X POST https://api.skillsafe.ai/v1/scan/github \
     -H "content-type: application/json" \
     -d '{"url": "https://github.com/OWNER/REPO", "subpath": "skills/pdf"}'
   ```

   Leave out `subpath` to scan every skill in the repository.

3. **Read the result.** `data.skills` has one entry per skill found, with:
   - `skill_name` and `subpath`
   - `grade` (`A+` best to `F` worst) and `score` (0–100)
   - `findings_count`
   - `skill_page_url`, a path on `https://skillsafe.ai` with the full report

   If `data.status` is `"failed"`, the scan did not complete: show the user
   `data.error` and do not treat the skill as safe. An HTTP 429 means the
   scanner is busy; wait a minute and retry once. An empty `data.skills` means
   no `SKILL.md` was found at that location.

4. **Report to the user, per skill:**
   - **A+ or A, no findings:** no known risky patterns were found.
   - **B or C:** there are findings to review. Many are capability flags
     (network access, running subprocesses) that can be legitimate; check
     whether they match what the skill says it does.
   - **D or F:** recommend not installing unless the user has read every
     finding and still wants to go ahead.
   - If `grade` is missing, judge by `findings_count` and the full report.

   Always include the full report link: `https://skillsafe.ai` followed by
   `skill_page_url`. It lists each finding and the skill's bill of materials
   (files, network destinations, environment variables, commands).

5. **Let the user decide.** This skill only scans; do not install anything as
   part of it.

## Notes

- A scan is point-in-time. Re-scan when the repository changes.
- A clean grade means no known risky patterns were found, not that the skill
  is guaranteed safe. The detection rules are published at
  https://skillsafe.ai/security/.
