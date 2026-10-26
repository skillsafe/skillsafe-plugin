---
name: deploy-app
description: Turn the user's skills or an idea into a hosted web app at https://SLUG.skillsafe.ai, either private for their own use or shared with specific people, anyone with the link, or the public directory. Use when the user wants to deploy, host, or publish an app, run a skill in the browser or from their phone, or let people without an agent use what they built.
---

# Deploy an app on SkillSafe

A SkillSafe app is a small hosted web app at `https://SLUG.skillsafe.ai/`: a
page the user designs, backed by an AI agent prompt that usually comes from
their skills. It runs in any browser, so it works for people with no agent
and from a phone.

The full, current reference for every step below is
**https://deploy.skillsafe.ai/llms.txt**. Fetch it and read the sections
"Visibility", "Deploying an app", and "App Spec v1" before building the
frontend; it is written for agents and covers formats and limits this skill
does not repeat.

## 0. Sign in, or deploy without an account

Run step 1 of `skillsafe-sign-in` to check for a stored key. If there is none,
offer the user the choice:

- **Sign in** (step 2 there): the app is permanent and they can manage it at
  https://skillsafe.ai.
- **Deploy now, sign up later** (step 3 there, a temporary key): no account
  needed to get a live link. Before deploying, tell them that the app is
  **deleted after 7 days unless they verify an email**, that its AI features
  run only after they verify (that is when the free credits arrive), and that
  inviting people and public listing also need verification. A temporary
  account can hold one app.

All calls below use this helper. It sends one request with the stored key and
prints the JSON response. Pass the method, the path, and optionally a JSON
body:

```bash
python3 - METHOD /v1/PATH '{"optional": "json body"}' <<'EOF'
import json, os, sys, urllib.request, urllib.error
cfg = json.load(open(os.path.expanduser("~/.skillsafe/config.json")))
method, path = sys.argv[1], sys.argv[2]
body = sys.argv[3].encode() if len(sys.argv) > 3 else None
req = urllib.request.Request(cfg.get("api_base", "https://api.skillsafe.ai") + path, data=body, method=method,
                             headers={"Authorization": f"Bearer {cfg['api_key']}", "content-type": "application/json"})
try:
    with urllib.request.urlopen(req, timeout=180) as r:
        print(r.status, r.read().decode())
except urllib.error.HTTPError as e:
    print(e.code, e.read().decode())
EOF
```

For a large body (a frontend release), write it to a file and pass
`"$(cat release.json)"` as the third argument, or adapt the helper to read the
file.

## 1. Decide who it is for

Ask the user. The answer sets the app's visibility:

| Who should use it | Visibility | How |
|---|---|---|
| Only the user | `private` | Set when creating. Nobody else can open it. |
| Specific people | `shared` | Create `private`, then `PATCH` to `shared` and invite emails (step 5). Invitees sign in with that address. |
| Anyone with the link | `unlisted` | Not listed anywhere, but anyone who has the URL can use it. Add `"require_login": true` to require a SkillSafe sign-in. |
| Everyone | `public` | Listed in the SkillSafe app directory. Requires a clean security scan. |

**Always create the app `private` first** and widen it only after the user has
tried it.

**Exception, temporary key:** a private app opens only for its owner signed in
at skillsafe.ai, which a temporary account cannot do, so create it `unlisted`
instead and tell the user that anyone with the link can open it until they
verify their email and switch it to `private`. Do not share the link around
in the meantime. `shared` invitations and `public` listing are refused on a
temporary key (`temp_key_scope`, `provisional_cannot_publish`) until the
email is verified.

## 2. Choose the agent prompt

- **From the user's skills (best):** the skills must be in their SkillSafe
  account. If they are only on this machine, back them up first with the
  `back-up-skills` skill. Then get each skill's id:
  `GET /v1/skills/@USERNAME/SKILL_NAME` → `data.skill_id`. Pass up to 8 as
  `skill_ids`, in the order the steps happen.
- **From a prompt you write, or a local skill's text:** pass `system_prompt`
  (SKILL.md format). Use this on a temporary key: paste the contents of the
  user's local `SKILL.md` files, since backing skills up needs a verified
  account.

An app is built for one **goal** ("turn a PDF into a one-page summary"); the
skills are steps toward it. Agree on the goal with the user before building.

## 3. Create the app

Run the step 0 helper with these arguments:

```
POST /v1/apps '{"slug": "pdf-summary", "title": "PDF Summary", "description": "Turn a PDF into a one-page summary.", "visibility": "private", "skill_ids": ["skl_..."]}'
```

The slug becomes the address (`https://pdf-summary.skillsafe.ai/`): lowercase
letters, digits, and hyphens. If it is taken, pick another.

## 4. Build and upload the frontend

Design the page for the goal: what does the user have to supply that the app
cannot work out itself? Then upload it as a release
(`POST /v1/apps/SLUG/releases`). Two options, both described in the reference:

- **App Spec** (`{"spec": {...}}`): a validated form the platform draws for
  you. Quickest for a form-in, result-out app.
- **Files** (`{"files": [{"path": "index.html", "content": "..."}, ...]}`):
  your own HTML, CSS, and JavaScript, for anything custom.

Rules that most often fail an upload: no inline `<script>` or `onclick=`
attributes (put all JavaScript in `.js` files), no third-party script CDNs
(include libraries as files), and no secrets in files. Every release is
security-scanned; a critical or high finding blocks it, and the error names
the file and line to fix.

## 5. Check it, then share it

1. `GET /v1/apps/SLUG` to confirm the release is live, then give the user the
   link `https://SLUG.skillsafe.ai/` and have them try it.
2. Widen access only when they are happy with it:
   - **Specific people:** `PATCH /v1/apps/SLUG` with `{"visibility": "shared"}`,
     then `POST /v1/apps/SLUG/share-grants` with `{"emails": ["a@example.com"]}`
     (up to 50; each gets an invitation email). Revoke with
     `DELETE /v1/apps/SLUG/share-grants/EMAIL`.
   - **Anyone with the link:** `PATCH` with `{"visibility": "unlisted"}`.
   - **Public directory:** `PATCH` with `{"visibility": "public"}`.
3. Confirm every change with a follow-up `GET`.

## 6. If the app was deployed with a temporary key

End every deploy on a temporary key with this reminder, using the date that
step 1 of `skillsafe-sign-in` prints:

> Your app is live at https://SLUG.skillsafe.ai/, but it is on a temporary
> account that will be **deleted on DATE**. Verify your email to keep it: it
> takes a minute, keeps the app at the same address, unlocks its AI features
> with free starting credits, and lets you make it private or invite people.
> Want to do that now?

If they say yes, follow step 4 of `skillsafe-sign-in`. Afterwards, offer to
switch the app to `private` (or `shared` with invitations) with
`PATCH /v1/apps/SLUG`. If they say not now, repeat the reminder (with the
days left) the next time any SkillSafe skill runs; step 1 of
`skillsafe-sign-in` reports it.

**Never run the browser sign-in instead of verifying** while the temporary
app should be kept: it stores a different key and leaves the app behind on
the temporary account, to be deleted.

## Costs

Running the app's AI uses SkillSafe credits, paid by the person running it:
the user for a private app; each signed-in user for a shared one. New accounts
get 3,000 free credits ($0.30) plus a daily grant for active users, so small
runs need no card. The owner can pay for other people's runs within daily
limits ("Sponsored usage" in the reference). Apps with no AI runs can be
completely free. Tell the user this before sharing an app with others.

## Change or remove an app

- **Update:** upload a new release. To change a few files, send
  `{"base": "current", "files": [...]}` so the rest carry over.
- **Roll back:** `GET /v1/apps/SLUG/releases`, then
  `POST /v1/apps/SLUG/releases/RELEASE_ID/promote`.
- **List the user's apps:** `GET /v1/apps`.
- **Delete:** only when the user asks, after confirming the slug:
  `DELETE /v1/apps/SLUG`.

The user can also manage apps in the dashboard at
https://skillsafe.ai/dashboard/.
