---
name: Prefer git switch over git checkout
description: When moving between branches, use `git switch` instead of `git checkout`
type: feedback
originSessionId: 83a78ce2-2a4e-4cb6-818e-6b7829c2f98b
---
Use `git switch <branch>` (and `git switch -c <branch>` for create) instead of `git checkout <branch>` when changing branches.

**Why:** User preference — `switch` is the modern, purpose-specific command for branch changes; `checkout` is overloaded.

**How to apply:** Any time branch navigation is needed in this project, propose and run `git switch`. Reserve `git checkout` only for its non-branch uses (e.g. `git checkout -- <file>` when explicitly needed).
