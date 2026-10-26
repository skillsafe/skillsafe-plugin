---
name: skillsafe-sign-in
description: Connect the agent to the user's SkillSafe account (browser sign-in, or a temporary key to deploy right away), store the key for the other SkillSafe skills, and turn a temporary account into a permanent one by verifying an email. Use before backing up skills or deploying an app when no SkillSafe key is stored, when a stored key is temporary, or when the user asks to sign in, log in, sign up, or claim their SkillSafe account.
---

# Sign in to SkillSafe

The other SkillSafe skills (`back-up-skills`, `deploy-app`) need an API key.
This skill gets one and stores it in `~/.skillsafe/config.json`, readable only
by the user. Never print the key, paste it into chat, or write it anywhere else.

There are two kinds of key:

- **Account key** (browser sign-in): permanent. Needed for backups.
- **Temporary key** (no sign-up): for deploying an app right away. The account
  and everything made with it are **deleted after 7 days** unless the user
  verifies an email (step 4), which keeps everything and makes it permanent.

## 1. Check for a stored key

Run this at the start of every SkillSafe task:

```bash
python3 - <<'EOF'
import json, os, urllib.request, urllib.error
path = os.path.expanduser("~/.skillsafe/config.json")
try:
    cfg = json.load(open(path))
except (OSError, ValueError):
    raise SystemExit("NO_KEY")
req = urllib.request.Request(cfg.get("api_base", "https://api.skillsafe.ai") + "/v1/account",
                             headers={"Authorization": f"Bearer {cfg['api_key']}"})
try:
    with urllib.request.urlopen(req, timeout=30) as r:
        acct = json.load(r)["data"]
except urllib.error.HTTPError as e:
    raise SystemExit(f"KEY_REJECTED {e.code}")
caps = acct.get("capabilities") or {}
if caps.get("credential") == "temp":
    print(f"TEMPORARY as {acct.get('username')}: deleted on {str(caps.get('expires_at'))[:10]} "
          f"({caps.get('days_remaining')} days left) unless an email is verified")
else:
    print("SIGNED_IN as", acct.get("username"))
EOF
```

- `SIGNED_IN`: done.
- `TEMPORARY`: usable for `deploy-app`. **Remind the user** of the deletion
  date and offer step 4 now. For backups, do step 4 first.
- `NO_KEY` or `KEY_REJECTED` (401, for example an expired temporary key):
  ask the user which they want. Use step 2 if they have or will create an
  account, or step 3 if they only want to deploy an app right now. After
  `KEY_REJECTED`, delete `~/.skillsafe/config.json` before step 3.

## 2. Browser sign-in (account key)

Tell the user you are opening a SkillSafe sign-in page; signing up is free
(Google, GitHub, or email). This opens the page, waits for approval (up to
15 minutes), and saves the key:

```bash
python3 - <<'EOF'
import json, os, socket, sys, time, urllib.request, urllib.error, webbrowser
API = "https://api.skillsafe.ai"
def post(path, body):
    req = urllib.request.Request(API + path, data=json.dumps(body).encode(), method="POST",
                                 headers={"content-type": "application/json"})
    with urllib.request.urlopen(req, timeout=30) as r:
        return json.load(r)["data"]
session = post("/v1/auth/cli", {"label": f"agent on {socket.gethostname()}"[:50]})
url = session["login_url"]
print("Open this link to approve the sign-in:", url, flush=True)
webbrowser.open(url)
deadline = time.time() + session.get("expires_in", 900)
while time.time() < deadline:
    time.sleep(10)
    try:
        with urllib.request.urlopen(f"{API}/v1/auth/cli/{session['session_id']}", timeout=30) as r:
            data = json.load(r)["data"]
    except urllib.error.HTTPError as e:
        sys.exit(f"Sign-in session ended (HTTP {e.code}). Run this step again.")
    if data.get("status") == "approved":
        os.makedirs(os.path.expanduser("~/.skillsafe"), mode=0o700, exist_ok=True)
        path = os.path.expanduser("~/.skillsafe/config.json")
        fd = os.open(path, os.O_WRONLY | os.O_CREAT | os.O_TRUNC, 0o600)
        with os.fdopen(fd, "w") as f:
            json.dump({"api_base": API, "api_key": data["api_key"],
                       "username": data["username"], "namespace": data["namespace"]}, f, indent=2)
        os.chmod(path, 0o600)
        print("SIGNED_IN as", data["username"])
        break
else:
    sys.exit("Timed out waiting for approval. Run this step again when the user is ready.")
EOF
```

- Show the user the printed link in case the browser did not open.
- The approval is read **once**: after approving, do not reopen or refresh the
  sign-in page, or the page consumes the key instead.
- Do not poll faster than every 10 seconds. HTTP 410 means the session
  expired; run this step again.

**Never replace a temporary key with this step** while the temporary account
holds an app the user wants to keep: that app stays on the temporary account
and is deleted. Use step 4 instead; it keeps everything.

## 3. Temporary key (deploy now, sign up later)

No account and no browser needed. Limited to 3 per hour per network, so mint
one and keep it:

```bash
python3 - <<'EOF'
import json, os, urllib.request
req = urllib.request.Request("https://api.skillsafe.ai/v1/auth/temp-key", data=b"", method="POST")
with urllib.request.urlopen(req, timeout=30) as r:
    data = json.load(r)["data"]
os.makedirs(os.path.expanduser("~/.skillsafe"), mode=0o700, exist_ok=True)
path = os.path.expanduser("~/.skillsafe/config.json")
if os.path.exists(path):
    raise SystemExit("A key is already stored; check it with step 1 instead of replacing it.")
fd = os.open(path, os.O_WRONLY | os.O_CREAT | os.O_EXCL, 0o600)
with os.fdopen(fd, "w") as f:
    json.dump({"api_base": "https://api.skillsafe.ai", "api_key": data["api_key"],
               "username": data["username"], "namespace": data["namespace"],
               "temporary": True, "expires_at": data["expires_at"]}, f, indent=2)
print("TEMPORARY as", data["username"], "| deleted on", data["expires_at"][:10], "unless an email is verified")
EOF
```

Tell the user plainly, before deploying anything:

- the app and account are deleted on the printed date unless they verify an
  email (step 4), which takes a minute and keeps everything;
- until then, AI features in the app cannot run: the free starting credits
  arrive with verification;
- inviting people and listing the app publicly also unlock with verification.

## 4. Verify an email (make a temporary account permanent)

Ask the user for their email address. If it already has a SkillSafe account,
the app and anything else made with the temporary key move into that account;
otherwise the temporary account becomes their new account. Either way the same
key keeps working.

Send the code:

```bash
python3 - "user@example.com" <<'EOF'
import json, os, sys, urllib.request
cfg = json.load(open(os.path.expanduser("~/.skillsafe/config.json")))
req = urllib.request.Request(cfg.get("api_base", "https://api.skillsafe.ai") + "/v1/account/email/send-code",
                             data=json.dumps({"email": sys.argv[1]}).encode(), method="POST",
                             headers={"Authorization": f"Bearer {cfg['api_key']}", "content-type": "application/json"})
with urllib.request.urlopen(req, timeout=30) as r:
    print("CODE_SENT; it expires in", json.load(r)["data"].get("expires_in"), "seconds")
EOF
```

Ask the user for the 6-digit code from the email, then confirm it. This also
updates the stored username, which changes if the app moved into an existing
account:

```bash
python3 - "user@example.com" "123456" <<'EOF'
import json, os, sys, time, urllib.request
path = os.path.expanduser("~/.skillsafe/config.json")
cfg = json.load(open(path))
api, auth = cfg.get("api_base", "https://api.skillsafe.ai"), {"Authorization": f"Bearer {cfg['api_key']}"}
req = urllib.request.Request(api + "/v1/account/email/verify-code",
                             data=json.dumps({"email": sys.argv[1], "code": sys.argv[2]}).encode(), method="POST",
                             headers={**auth, "content-type": "application/json"})
with urllib.request.urlopen(req, timeout=30) as r:
    result = json.load(r)["data"]
cfg.update(username=result["username"], namespace="@" + result["username"])
cfg.pop("temporary", None); cfg.pop("expires_at", None)
fd = os.open(path, os.O_WRONLY | os.O_TRUNC, 0o600)
with os.fdopen(fd, "w") as f:
    json.dump(cfg, f, indent=2)
print("VERIFIED as", result["username"], "| moved into existing account" if result.get("transferred") else "| new account")
EOF
```

A wrong or expired code returns an error; send a new code and try again.
Allow about 30 seconds before the key's new permissions take effect. From now
on the user can also sign in at https://skillsafe.ai with that email to manage
their apps.

## Signing out

Delete `~/.skillsafe/config.json`. To revoke the key itself, the user can remove
it at https://skillsafe.ai/account/keys.
