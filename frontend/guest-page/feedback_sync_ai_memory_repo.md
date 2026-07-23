---
name: Memory base is the corepark-ai-memory repo
description: The authoritative memory for this project lives in /Users/israel/Dev/corepark-ai-memory/frontend/guest-page/. Read from there, write to there. Free to git add/commit/push in this repo — it is private and internal to Corepark, meant to accumulate learnings across features.
type: feedback
originSessionId: 9b60e26d-792d-4aa4-9aab-212bea254a10
---
The base of this project's memory is the git-tracked repo at **`/Users/israel/Dev/corepark-ai-memory/frontend/guest-page/`**. Treat it as the source of truth.

**Why:** The repo is a private, internal-to-Corepark knowledge base that accumulates everything learned from every feature. Files that live only in the per-session auto-memory folder (`~/.claude/projects/-Users-israel-Dev-frontend-guest-page/memory/`) never leave this machine and are invisible to teammates or other Claude sessions. Getting knowledge into the repo is what makes it useful.

**How to apply:**
- **Read:** when checking existing memory, prefer the repo path. If a file appears only in the local folder, treat it as stale/unofficial.
- **Write:** when creating or updating a memory (including `MEMORY.md`), write **directly to the repo path**. Also mirror to the local auto-memory folder so it injects on next session start. Same-content 1:1 by filename.
- **Delete:** removals happen in the repo first, then mirror to local.
- **Git actions are permitted in this repo.** Unlike the code project (where the commit-workflow rule forbids running git commands), inside `/Users/israel/Dev/corepark-ai-memory` you may run `git add`, `git commit`, and `git push` yourself once a memory task is done. Write a clear conventional-commit message, run the commands, and report the result. The user has granted blanket permission because the repo is private and low-risk.
- **Scope of the git permission:** applies **only** to `/Users/israel/Dev/corepark-ai-memory`. Every other repo (including `frontend-guest-page`) still follows the standard commit-workflow rule — provide commit message text only, do not run git.
- **After every write, verify with `diff`** to confirm the repo and local copies match.
