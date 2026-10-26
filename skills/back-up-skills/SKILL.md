---
name: back-up-skills
description: Back up the user's personal agent skills to their private SkillSafe account with version history, list what is backed up, and restore skills on this or another machine. Use when the user wants to back up, save, version, sync, move, or restore their skills, set up their skills on a new computer, or delete a backup.
---

# Back up and manage personal skills

SkillSafe keeps the user's skills in their account: **private** (only they can
see them), versioned (every backup is a new version; older ones stay
available), and restorable on any machine where they sign in.

**Before anything else**, run step 1 of the `skillsafe-sign-in` skill.
Backups need a permanent account:

- no key stored: sign in through the browser (step 2 there);
- a **temporary** key (from deploying an app without an account): have the
  user verify their email first (step 4 there). That makes the account
  permanent and keeps their app. Never back up to a temporary account: it is
  deleted after 7 days with everything in it.

**What this sends:** the files of each skill being backed up, to
`https://api.skillsafe.ai`, stored privately in the user's account. Files named
`.env*`, `.DS_Store`, and the folders `.git`, `node_modules`, `__pycache__`,
`.venv`, `venv` are never sent. If a skill contains a real secret (an API key,
token, or password in a file), tell the user and leave that file out or have
them remove the secret first.

## Find the skills

A skill is a folder containing `SKILL.md`. Ask the user which skills to back
up, or offer the ones found in the usual places:

- this agent's personal skills folder (for example `~/.agents/skills/`,
  `~/.codex/skills/`, or `~/.gemini/skills/`)
- the current project's skills folder (for example `.agents/skills/`)

```bash
find ~/.agents/skills ~/.codex/skills ~/.gemini/skills .agents/skills \
  -maxdepth 2 -name SKILL.md 2>/dev/null
```

Use the skill's `name` from its `SKILL.md` front matter as the backup name
(lowercase letters, digits, and hyphens). Skip agent-managed system folders such
as `.system`.

## Back up a skill

Run once per skill, with the skill folder and backup name. It saves a new
version only if something changed:

```bash
python3 - "PATH/TO/SKILL_FOLDER" "skill-name" <<'EOF'
import hashlib, json, os, pathlib, sys, urllib.request, urllib.error, uuid
cfg = json.load(open(os.path.expanduser("~/.skillsafe/config.json")))
API, KEY, NS = cfg.get("api_base", "https://api.skillsafe.ai"), cfg["api_key"], cfg["username"]
folder, name = pathlib.Path(sys.argv[1]).expanduser().resolve(), sys.argv[2]
def call(method, path, body=None, ctype="application/json"):
    req = urllib.request.Request(API + path, data=body, method=method,
                                 headers={"Authorization": f"Bearer {KEY}", "content-type": ctype})
    try:
        with urllib.request.urlopen(req, timeout=120) as r:
            return r.status, json.load(r)
    except urllib.error.HTTPError as e:
        return e.code, json.loads(e.read() or b"{}")
skip = {".git", "node_modules", "__pycache__", ".venv", "venv", ".skillsafe"}
files = {}
for p in sorted(folder.rglob("*")):
    rel = p.relative_to(folder)
    if p.is_file() and not skip & set(rel.parts) and not p.name.startswith(".env") and p.name != ".DS_Store":
        files[rel.as_posix()] = p.read_bytes()
if "SKILL.md" not in files:
    sys.exit(f"No SKILL.md in {folder}")
manifest = [{"path": k, "hash": "sha256:" + hashlib.sha256(v).hexdigest(), "size": len(v)} for k, v in files.items()]
# Next version: 1.0.0 for a new backup, else bump the patch number; skip if unchanged.
st, meta = call("GET", f"/v1/skills/@{NS}/{name}")
latest = meta.get("data", {}).get("latest_version") if st == 200 else None
if latest:
    _, ver = call("GET", f"/v1/skills/@{NS}/{name}/versions/{latest}")
    old = {(f.get("file_path") or f.get("path"), f.get("file_hash") or f.get("hash")) for f in ver.get("data", {}).get("files", [])}
    if old == {(m["path"], m["hash"]) for m in manifest}:
        sys.exit(f"UNCHANGED @{NS}/{name} {latest}")
    major, minor, patch = (int(x) for x in latest.split("-")[0].split(".")[:3])
else:
    major, minor, patch = 1, 0, -1
for attempt in range(5):  # a 409 means that version already exists: take the next one
    patch += 1
    version = f"{major}.{minor}.{patch}"
    st, neg = call("POST", f"/v1/skills/@{NS}/{name}/negotiate",
                   json.dumps({"version": version, "file_manifest": manifest}).encode())
    if st == 409:
        continue
    if st != 200:
        sys.exit(f"negotiate failed ({st}): {neg.get('error')}")
    needed = set(neg["data"]["needed_files"])
    boundary = uuid.uuid4().hex
    parts = [f'--{boundary}\r\nContent-Disposition: form-data; name="metadata"\r\n\r\n'.encode()
             + json.dumps({"version": version, "file_manifest": manifest}).encode() + b"\r\n"]
    for i, path in enumerate(sorted(needed)):
        parts.append(f'--{boundary}\r\nContent-Disposition: form-data; name="file_{i}"; filename="{path}"\r\n'
                     f'Content-Type: application/octet-stream\r\n\r\n'.encode() + files[path] + b"\r\n")
    parts.append(f"--{boundary}--\r\n".encode())
    st, res = call("POST", f"/v1/skills/@{NS}/{name}", b"".join(parts), f"multipart/form-data; boundary={boundary}")
    if st == 409:
        continue
    if st not in (200, 201):
        sys.exit(f"backup failed ({st}): {res.get('error')}")
    print(f"BACKED_UP @{NS}/{name} {version} ({len(manifest)} files, {len(needed)} uploaded)")
    break
else:
    sys.exit("backup failed: could not find a free version number; list the versions and retry")
EOF
```

Report each result to the user. `UNCHANGED` is success: the latest backup
already matches. Common failures:

- **413 / storage limit:** the account's storage is full (free accounts get
  50 MB). Leave out large generated files, or the user can upgrade at
  https://skillsafe.ai/pricing/.
- **429:** too many saves in 24 hours (50 on the free plan). Try later.
- **400 (older than latest):** a newer backup was saved from another
  machine; run the backup again.

## List backups

```bash
python3 - <<'EOF'
import json, os, urllib.request
cfg = json.load(open(os.path.expanduser("~/.skillsafe/config.json")))
API = cfg.get("api_base", "https://api.skillsafe.ai")
req = urllib.request.Request(f"{API}/v1/account/skills?limit=100",
                             headers={"Authorization": f"Bearer {cfg['api_key']}"})
with urllib.request.urlopen(req, timeout=30) as r:
    for s in json.load(r)["data"]:
        print(f"{s['namespace']}/{s['name']}  v{s.get('latest_version')}  {s.get('visibility')}  updated {str(s.get('updated_at'))[:10]}")
EOF
```

To see every saved version of one skill:
`GET https://api.skillsafe.ai/v1/skills/@USERNAME/NAME/versions` with the same
`Authorization` header.

## Restore a skill

Restores the latest version, or a given one, into a folder. Ask before
overwriting a folder that already exists; restore into the agent's personal
skills folder unless the user wants it elsewhere.

```bash
python3 - "skill-name" "DESTINATION/skill-name" "latest" <<'EOF'
import hashlib, json, os, pathlib, sys, urllib.request
cfg = json.load(open(os.path.expanduser("~/.skillsafe/config.json")))
API, KEY, NS = cfg.get("api_base", "https://api.skillsafe.ai"), cfg["api_key"], cfg["username"]
name, dest, version = sys.argv[1], pathlib.Path(sys.argv[2]).expanduser(), sys.argv[3]
def get(path):
    req = urllib.request.Request(API + path, headers={"Authorization": f"Bearer {KEY}"})
    with urllib.request.urlopen(req, timeout=120) as r:
        return r.read()
if version == "latest":
    version = json.loads(get(f"/v1/skills/@{NS}/{name}"))["data"]["latest_version"]
listing = json.loads(get(f"/v1/skills/@{NS}/{name}/download/{version}"))["data"]
for f in listing["files"]:
    target = (dest / f["path"]).resolve()
    if not str(target).startswith(str(dest.resolve()) + os.sep):
        sys.exit(f"Refusing unsafe path {f['path']}")
    data = get(f"/v1/blobs/{f['hash']}")
    if "sha256:" + hashlib.sha256(data).hexdigest() != f["hash"]:
        sys.exit(f"Integrity check failed for {f['path']}")
    target.parent.mkdir(parents=True, exist_ok=True)
    target.write_bytes(data)
print(f"RESTORED @{NS}/{name} {version} -> {dest} ({len(listing['files'])} files)")
EOF
```

Every file is checked against the hash recorded at backup time before it is
written. After restoring, tell the user to start a new agent session so the
skill is picked up.

To set up a new machine: sign in there (`skillsafe-sign-in`), list the
backups, and restore the ones the user wants.

## Delete a backup

Only when the user explicitly asks, and after confirming the exact skill name:
it removes the skill and all of its versions from their account.

```bash
python3 - "skill-name" <<'EOF'
import json, os, sys, urllib.request
cfg = json.load(open(os.path.expanduser("~/.skillsafe/config.json")))
API, NS = cfg.get("api_base", "https://api.skillsafe.ai"), cfg["username"]
req = urllib.request.Request(f"{API}/v1/skills/@{NS}/{sys.argv[1]}", method="DELETE",
                             headers={"Authorization": f"Bearer {cfg['api_key']}"})
with urllib.request.urlopen(req, timeout=30) as r:
    print("DELETED", f"@{NS}/{sys.argv[1]}", r.status)
EOF
```

## Good to know

- Backups are private. Nothing is shared or published unless the user does
  that separately on https://skillsafe.ai.
- The user can browse and manage the same backups in the web dashboard
  (https://skillsafe.ai/dashboard/) or the AI SkillSafe desktop app
  (https://app.skillsafe.ai/).
- A backed-up skill can be turned into a hosted app with the `deploy-app` skill.
