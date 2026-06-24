# Skill: github-api-push

## Purpose
Push files (any type: `.md`, `.png`, `.pdf`, `.py`, etc.) to a GitHub repository using the GitHub REST API v3 — no git CLI, no MCP connector, no approval dialogs. Works entirely via `urllib` (Python stdlib, zero dependencies).

---

## Credentials

| Variable | Description |
|---|---|
| `GITHUB_TOKEN` | Personal Access Token — scope: `repo` (full control of private repos) |
| `GITHUB_OWNER` | Repository owner or org name (e.g. `burcakozkok`) |
| `GITHUB_REPO`  | Repository name (e.g. `chronos`) |

**Generate a PAT:** GitHub → Settings → Developer settings → Personal access tokens → Tokens (classic) → Generate new token → check `repo` scope.

Set as environment variables or pass as `--token / --owner / --repo` flags.

---

## Core Pattern

Every file push is a single HTTP `PUT` request to:

```
PUT https://api.github.com/repos/{owner}/{repo}/contents/{path}
```

**Required fields:**
```json
{
  "message": "commit message",
  "content": "<base64-encoded file bytes>",
  "branch":  "main",
  "sha":     "<current blob SHA — required only when updating an existing file>"
}
```

**Get the current SHA** before updating (returns `null` / 404 if file is new):
```
GET https://api.github.com/repos/{owner}/{repo}/contents/{path}?ref={branch}
```

---

## Canonical Python Implementation

```python
import json, base64, urllib.request, urllib.error, os

TOKEN  = os.environ["GITHUB_TOKEN"]
OWNER  = os.environ["GITHUB_OWNER"]
REPO   = os.environ["GITHUB_REPO"]
BRANCH = "main"

def get_sha(remote_path: str) -> str | None:
    """Return existing blob SHA, or None if file does not exist yet."""
    url = f"https://api.github.com/repos/{OWNER}/{REPO}/contents/{remote_path}?ref={BRANCH}"
    req = urllib.request.Request(url)
    req.add_header("Authorization", f"token {TOKEN}")
    req.add_header("Accept", "application/vnd.github.v3+json")
    try:
        with urllib.request.urlopen(req) as r:
            return json.load(r)["sha"]
    except urllib.error.HTTPError as e:
        if e.code == 404:
            return None
        raise

def push_file(local_path: str, remote_path: str, message: str):
    """Create or update a file in the repo."""
    with open(local_path, "rb") as f:
        encoded = base64.b64encode(f.read()).decode("utf-8")

    payload = {"message": message, "content": encoded, "branch": BRANCH}
    sha = get_sha(remote_path)
    if sha:
        payload["sha"] = sha          # required for updates; omit for new files

    url = f"https://api.github.com/repos/{OWNER}/{REPO}/contents/{remote_path}"
    req = urllib.request.Request(url, data=json.dumps(payload).encode(), method="PUT")
    req.add_header("Authorization", f"token {TOKEN}")
    req.add_header("Accept", "application/vnd.github.v3+json")
    req.add_header("Content-Type", "application/json")

    with urllib.request.urlopen(req) as r:
        result = json.load(r)
        print("✅", result["content"]["html_url"])
        print("   Commit:", result["commit"]["sha"])
```

**Usage:**
```python
push_file(
    local_path  = "/tmp/output.md",
    remote_path = "01_chapter1/output.md",
    message     = "enrich Chapter 1 with new sections"
)
```

---

## Pushing Binary Files (images, PDFs)

Identical pattern — `base64.b64encode` handles all binary formats transparently.

```python
push_file(
    local_path  = "/tmp/cover.png",
    remote_path = "01_chapter1/cover.png",
    message     = "add Ghibli cover illustration"
)
```

---

## Pushing Large Files (> 1 MB)

The Contents API has a **1 MB soft limit** per file. For larger files (e.g. high-res images, PDFs), use the Git Data API instead:

```
POST /repos/{owner}/{repo}/git/blobs   → get blob SHA
POST /repos/{owner}/{repo}/git/trees   → attach blob to tree
POST /repos/{owner}/{repo}/git/commits → create commit
PATCH /repos/{owner}/{repo}/git/refs/heads/{branch} → advance branch pointer
```

For files under ~3 MB, the Contents API generally works in practice despite the stated limit.

---

## CLI Tool (github_push.py)

A ready-to-use command-line wrapper is available in this project as `github_push.py`.

```bash
# Set credentials
export GITHUB_TOKEN=ghp_...
export GITHUB_OWNER=burcakozkok
export GITHUB_REPO=chronos

# Push one file
python github_push.py --folder 01_chapter1 output.md

# Push multiple files in one commit
python github_push.py --folder 01_chapter1 output.md cover.png \
    --message "add Chapter 1 final draft + illustration"

# Preview without pushing
python github_push.py --folder 02_chapter2 output.md --dry-run

# Override credentials inline
python github_push.py \
    --token ghp_xxx --owner alice --repo mybook \
    --folder 03_chapter3 output.md
```

---

## Error Reference

| HTTP Code | Meaning | Fix |
|---|---|---|
| `401 Unauthorized` | Bad or expired token | Regenerate PAT in GitHub settings |
| `403 Forbidden` | Token lacks `repo` scope | Re-create token with `repo` checked |
| `404 Not Found` | Repo or path doesn't exist | Check `OWNER`, `REPO`, branch name |
| `409 Conflict` | SHA mismatch on update | Fetch fresh SHA with `get_sha()` before pushing |
| `422 Unprocessable` | Payload validation error | Check base64 encoding and JSON structure |

---

## Key Rules

1. **Always fetch SHA before updating.** Attempting to update a file without its current SHA returns `409 Conflict`.
2. **Omit SHA for new files.** Including an SHA when the file doesn't exist yet causes `422`.
3. **One file per API call.** The Contents API does not support batch commits. For multi-file atomic commits, use the Git Data API (blobs → tree → commit → ref).
4. **Binary = base64.** All content must be base64-encoded, including plain text files.
5. **Token scope = `repo`.** The `public_repo` scope is insufficient for pushing content even to public repos in some configurations.

---

## Quick Reference

```python
# Minimal push (new file)
import json, base64, urllib.request
payload = json.dumps({
    "message": "add file",
    "content": base64.b64encode(open("file.md","rb").read()).decode(),
    "branch":  "main"
}).encode()
req = urllib.request.Request(
    "https://api.github.com/repos/OWNER/REPO/contents/path/file.md",
    data=payload, method="PUT"
)
req.add_header("Authorization", "token GITHUB_TOKEN")
req.add_header("Accept", "application/vnd.github.v3+json")
req.add_header("Content-Type", "application/json")
with urllib.request.urlopen(req) as r:
    print(json.load(r)["content"]["html_url"])
```
