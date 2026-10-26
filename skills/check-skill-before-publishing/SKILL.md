---
name: check-skill-before-publishing
description: Security-check an agent skill the user has written before they share or publish it, using the free SkillSafe file scanner, and explain how to fix what it finds. Use when the user wants to publish, share, or distribute a skill, or asks whether their own skill is safe or well-formed.
---

# Check a skill before publishing it

Run the user's own skill through SkillSafe's free scanner, explain each
finding, help fix it, and then point the user to where they can publish. No
account or API key is needed for the scan.

**What this sends:** the text contents of the files in the skill folder, in one
request to `https://api.skillsafe.ai/v1/scan/files`. The scanner analyzes them
and returns the result; it does not store them. Tell the user this and get their
OK before sending, and never include secrets (see step 1).

## Steps

1. **Collect the skill's files.** Take the folder that contains `SKILL.md`.
   Include text files only. Leave out `.git/`, `node_modules/`, virtual
   environments, binary files (images, archives, executables), and any file that
   holds credentials, such as `.env` files or key files. If a file the skill
   needs contains a real secret, stop and tell the user to remove it first:
   a published skill must not contain one.

2. **Scan them.** Send a JSON body of the form
   `{"files": [{"path": "SKILL.md", "content": "..."}, ...]}`, with each `path`
   relative to the skill folder. For example, from inside the skill folder:

   ```bash
   python3 - <<'EOF' > /tmp/skill-scan.json
   import json, pathlib
   skip_dirs = {".git", "node_modules", ".venv", "venv", "__pycache__"}
   files = []
   for p in sorted(pathlib.Path(".").rglob("*")):
       if not p.is_file() or skip_dirs & set(p.parts) or p.name.startswith(".env"):
           continue
       try:
           files.append({"path": p.as_posix(), "content": p.read_text("utf-8")})
       except UnicodeDecodeError:
           continue  # binary file
   print(json.dumps({"files": files}))
   EOF
   curl -s -X POST https://api.skillsafe.ai/v1/scan/files \
     -H "content-type: application/json" --data @/tmp/skill-scan.json
   ```

   Limits: up to 1,000 files, 2 MB per file, 20 MB in total.

3. **Read the result.** `data.raw_findings` lists each finding with
   `severity`, `rule_id`, `file`, `line`, and `message`. `data.bom` is the bill
   of materials: the network destinations, environment variables, file access,
   and commands the scanner found.

4. **Explain and fix, by severity:**
   - **critical / high:** must be fixed before publishing. Typical causes:
     a hard-coded secret, decoding and running encoded content, sending
     environment variables to a server, writing to an agent's memory or
     instruction files.
   - **medium:** review with the user. Often a real capability that the skill
     does not mention: say in `SKILL.md` what the skill does and why (for
     example "this script calls api.example.com to fetch rates").
   - **low / info:** usually fine; mention them briefly.

   Undocumented capabilities are reported as `undoc_network`, `undoc_env_read`,
   `undoc_subprocess`, and `undoc_file_write`. They clear once `SKILL.md`
   describes the behavior. After fixing, scan again.

5. **Publish.** When nothing critical or high remains, the user can publish
   the skill on SkillSafe, which scans it again and shows the result on its
   page, from the AI SkillSafe desktop app (https://app.skillsafe.ai/) or the
   web dashboard (https://skillsafe.ai/dashboard/). Skills are saved privately
   first; sharing them publicly is a separate step that requires a clean scan.

## Authoring checklist

A skill that scans clean usually:

- documents every network call, environment variable, subprocess, and file
  write its scripts make;
- contains no API keys, tokens, or private keys, including in examples;
- never writes to an agent's instruction or memory files;
- contains no encoded payloads (base64 blobs are decoded and scanned too).
