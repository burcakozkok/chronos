# Skill: github-api-connect

## Purpose
Read from and write to a GitHub repository using the GitHub REST API v3 — no git CLI, no MCP connector, no approval dialogs. Works entirely via `urllib` (Python stdlib, zero dependencies). Covers the full cycle: pull files, list folders, read content, push new files, update existing files.

---

## Credentials

| Variable | Description |
|---|---|
| `GITHUB_TOKEN` | Personal Access Token — scope: `repo` (full control of private repos) |
| `GITHUB_OWNER` | Repository owner or org name (e.g. `burcakozkok`) |
| `GITHUB_REPO`  | Repository name (e.g. `chronos`) |

**Generate a PAT:** GitHub → Settings → Developer settings → Personal access tokens → Tokens (classic) → Generate new token → check `repo` scope.

Set as environment variables or pass explicitly in code.

```python
import os
TOKEN  = os.environ["GITHUB_TOKEN"]
OWNER  = os.environ["GITHUB_OWNER"]
REPO   = os.environ["GITHUB_REPO"]
BRANCH = "main"
```

---

## Core API Endpoint

All file operations use:
```
https://api.github.com/repos/{owner}/{repo}/contents/{path}
```

| Operation | Method | Notes |
|---|---|---|
| Read file / get SHA | `GET` | Returns metadata + base64 content |
| List folder | `GET` | Returns array of file/dir entries |
| Create new file | `PUT` | No `sha` field in payload |
| Update existing file | `PUT` | Must include current `sha` in payload |

---

## PULL — Reading from the Repo

### Helper: make an authenticated request

```python
import json, urllib.request, urllib.error

def gh_get(path: str) -> dict | list:
    """GET any GitHub API path. Returns parsed JSON."""
    url = f"https://api.github.com/repos/{OWNER}/{REPO}/contents/{path}?ref={BRANCH}"
    req = urllib.request.Request(url)
    req.add_header("Authorization", f"token {TOKEN}")
    req.add_header("Accept", "application/vnd.github.v3+json")
    with urllib.request.urlopen(req) as r:
        return json.load(r)
```

---

### Get file metadata + SHA

```python
def get_sha(remote_path: str) -> str | None:
    """Return current blob SHA for a file, or None if it does not exist."""
    try:
        return gh_get(remote_path)["sha"]
    except urllib.error.HTTPError as e:
        if e.code == 404:
            return None
        raise
```

---

### Read a text file

```python
import base64

def read_text_file(remote_path: str) -> str:
    """Download and decode a UTF-8 text file from the repo."""
    data = gh_get(remote_path)
    return base64.b64decode(data["content"]).decode("utf-8")

# Example
text = read_text_file("01_chapter1/output.md")
print(text[:500])
```

---

### Read a binary file (image, PDF, etc.)

```python
def read_binary_file(remote_path: str, save_to: str):
    """Download a binary file and save it locally."""
    data = gh_get(remote_path)
    raw = base64.b64decode(data["content"])
    with open(save_to, "wb") as f:
        f.write(raw)
    print(f"Saved {len(raw):,} bytes → {save_to}")

# Example
read_binary_file("01_chapter1/cover.png", "/tmp/cover.png")
```

---

### List all files in a folder

```python
def list_folder(remote_folder: str) -> list[dict]:
    """
    Return a list of entries in a repo folder.
    Each entry: {name, path, sha, size, type}  where type = 'file' or 'dir'
    """
    entries = gh_get(remote_folder)
    return [
        {
            "name": e["name"],
            "path": e["path"],
            "sha":  e["sha"],
            "size": e.get("size", 0),
            "type": e["type"],
        }
        for e in entries
    ]

# Example
for entry in list_folder("01_chapter1"):
    print(f"  {entry['type']:4}  {entry['size']:>8,} bytes  {entry['name']}")
```

---

### List all folders (top-level repo structure)

```python
def list_repo_root() -> list[dict]:
    """List top-level files and folders in the repo root."""
    return list_folder("")          # empty string = repo root

# Example
for entry in list_repo_root():
    print(f"  {entry['type']:4}  {entry['name']}")
```

---

## PUSH — Writing to the Repo

### Push any file (create or update automatically)

```python
def push_file(local_path: str, remote_path: str, message: str):
    """
    Create or update a file in the repo.
    Automatically fetches the current SHA when updating an existing file.
    Works for text and binary files (images, PDFs, etc.).
    """
    with open(local_path, "rb") as f:
        encoded = base64.b64encode(f.read()).decode("utf-8")

    payload = {"message": message, "content": encoded, "branch": BRANCH}

    sha = get_sha(remote_path)
    if sha:
        payload["sha"] = sha        # required for updates; omit for new files

    url = f"https://api.github.com/repos/{OWNER}/{REPO}/contents/{remote_path}"
    req = urllib.request.Request(url, data=json.dumps(payload).encode(), method="PUT")
    req.add_header("Authorization", f"token {TOKEN}")
    req.add_header("Accept", "application/vnd.github.v3+json")
    req.add_header("Content-Type", "application/json")

    with urllib.request.urlopen(req) as r:
        result = json.load(r)
        print("✅", result["content"]["html_url"])
        print("   Commit:", result["commit"]["sha"])

# Examples
push_file("/tmp/output.md",  "01_chapter1/output.md", "update Chapter 1 draft")
push_file("/tmp/cover.png",  "01_chapter1/cover.png", "add cover illustration")
```

---

### Push text content directly (no local file needed)

```python
def push_text(content: str, remote_path: str, message: str):
    """Push a string directly to the repo without writing a local file first."""
    encoded = base64.b64encode(content.encode("utf-8")).decode("utf-8")

    payload = {"message": message, "content": encoded, "branch": BRANCH}
    sha = get_sha(remote_path)
    if sha:
        payload["sha"] = sha

    url = f"https://api.github.com/repos/{OWNER}/{REPO}/contents/{remote_path}"
    req = urllib.request.Request(url, data=json.dumps(payload).encode(), method="PUT")
    req.add_header("Authorization", f"token {TOKEN}")
    req.add_header("Accept", "application/vnd.github.v3+json")
    req.add_header("Content-Type", "application/json")

    with urllib.request.urlopen(req) as r:
        result = json.load(r)
        print("✅", result["content"]["html_url"])
        print("   Commit:", result["commit"]["sha"])

# Example
push_text("# Hello\nThis file was written directly from memory.", "notes/hello.md", "add hello note")
```

---

## Combined Read → Modify → Write Pattern

The most common real-world workflow: pull a file, modify its content in memory, push the updated version back.

```python
def update_text_file(remote_path: str, modifier_fn, message: str):
    """
    Pull a text file, apply modifier_fn(text) -> new_text, push the result back.

    Args:
        remote_path : path in the repo (e.g. "01_chapter1/output.md")
        modifier_fn : callable that takes the current text and returns modified text
        message     : commit message
    """
    # 1. Pull
    data    = gh_get(remote_path)
    current = base64.b64decode(data["content"]).decode("utf-8")
    sha     = data["sha"]

    # 2. Modify
    updated = modifier_fn(current)

    # 3. Push
    encoded = base64.b64encode(updated.encode("utf-8")).decode("utf-8")
    payload = {"message": message, "content": encoded, "branch": BRANCH, "sha": sha}

    url = f"https://api.github.com/repos/{OWNER}/{REPO}/contents/{remote_path}"
    req = urllib.request.Request(url, data=json.dumps(payload).encode(), method="PUT")
    req.add_header("Authorization", f"token {TOKEN}")
    req.add_header("Accept", "application/vnd.github.v3+json")
    req.add_header("Content-Type", "application/json")

    with urllib.request.urlopen(req) as r:
        result = json.load(r)
        print("✅ Updated:", result["content"]["html_url"])

# Example: append a footnote to Chapter 1
update_text_file(
    remote_path = "01_chapter1/output.md",
    modifier_fn = lambda text: text + "\n\n---\n*Last updated by agent.*",
    message     = "append footnote to Chapter 1"
)

# Example: find-and-replace a section heading
update_text_file(
    remote_path = "01_chapter1/output.md",
    modifier_fn = lambda text: text.replace("## Looking Ahead", "## The Next Revolution"),
    message     = "rename final section heading"
)
```

---

## CLI Tool: github_push.py

A ready-to-use command-line wrapper for push operations is available as `github_push.py` in this project.

```bash
# Set credentials once
export GITHUB_TOKEN=ghp_...
export GITHUB_OWNER=burcakozkok
export GITHUB_REPO=chronos

# Push a single file
python github_push.py --folder 01_chapter1 output.md

# Push multiple files in one commit
python github_push.py --folder 01_chapter1 output.md cover.png \
    --message "Chapter 1 final draft + illustration"

# Dry-run preview
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
| `404 Not Found` | Repo, path, or branch doesn't exist | Check `OWNER`, `REPO`, `BRANCH`, path spelling |
| `409 Conflict` | SHA mismatch on update | Always fetch fresh SHA via `get_sha()` just before pushing |
| `422 Unprocessable` | Payload validation error | Check base64 encoding and JSON structure |

---

## Key Rules

1. **Always fetch SHA before updating.** Updating without the current SHA → `409 Conflict`.
2. **Omit SHA for new files.** Including a SHA when the file doesn't exist → `422`.
3. **One file per Contents API call.** For multi-file atomic commits, use the Git Data API (blobs → tree → commit → ref).
4. **All content must be base64-encoded** — including plain text files.
5. **Token scope must be `repo`** — `public_repo` is insufficient for push operations even on public repos.
6. **File size limit ~1 MB** via the Contents API. Larger files require the Git Blobs API.

---

## Quick Reference Card

```python
import json, base64, urllib.request, urllib.error, os

TOKEN, OWNER, REPO, BRANCH = (
    os.environ["GITHUB_TOKEN"], os.environ["GITHUB_OWNER"],
    os.environ["GITHUB_REPO"],  "main"
)
BASE = f"https://api.github.com/repos/{OWNER}/{REPO}/contents"

def _req(path, payload=None):
    req = urllib.request.Request(
        f"{BASE}/{path}",
        data=json.dumps(payload).encode() if payload else None,
        method="PUT" if payload else "GET"
    )
    req.add_header("Authorization", f"token {TOKEN}")
    req.add_header("Accept", "application/vnd.github.v3+json")
    if payload:
        req.add_header("Content-Type", "application/json")
    with urllib.request.urlopen(req) as r:
        return json.load(r)

# PULL: read text
def pull(path):
    return base64.b64decode(_req(path)["content"]).decode()

# PULL: get SHA
def sha(path):
    try:    return _req(path)["sha"]
    except urllib.error.HTTPError as e:
        if e.code == 404: return None
        raise

# PULL: list folder
def ls(folder=""):
    return [(e["name"], e["type"], e["size"]) for e in _req(folder)]

# PUSH: file or string
def push(content, path, msg):
    if isinstance(content, str):  content = content.encode()
    p = {"message": msg, "content": base64.b64encode(content).decode(), "branch": BRANCH}
    s = sha(path)
    if s: p["sha"] = s
    _req(path, p)
```
