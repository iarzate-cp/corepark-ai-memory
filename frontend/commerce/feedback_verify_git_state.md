---
name: Verify code exists in origin/main before claiming an API exists
description: When mapping cross-repo dependencies or reporting what a service exposes, don't trust file contents alone — a dirty working tree can make an unmerged endpoint look like it already ships
type: feedback
originSessionId: a2019106-3f08-4f27-8756-0bb019dfebd1
---
When exploring a repo to report what endpoints, methods, or APIs it exposes, verify the relevant code is committed to `origin/main` (or the equivalent deployable branch), not just present in the working tree.

**Why:** On 2026-07-06 I described the `DELETE /ticket` endpoint in `ms-firebase-service` as an existing API of the service. It was actually uncommitted work in the user's local tree — three modified files (controller, service impl, DAO impl) that had never been pushed. The Explore subagent read the modified files as-is without noticing the git state. This propagated into a false assumption that the ms-backoffice-service orphan-cleanup feature could ship independently, when in reality it depends on that ms-firebase-service PR being merged and deployed first. The user only caught this after running `git pull` and asking me to re-check the repo.

**How to apply:** Any time a report will inform a shipping decision or a cross-service integration contract, run at least a quick `git status` and `git log -1 -- <file>` on the files being cited. If a file is modified locally or its last commit is on a branch other than main/master, flag the cross-repo dependency explicitly in the report — do not silently treat working-tree state as canonical. Especially when tracing microservice-to-microservice contracts across repos, treat each endpoint as a claim that must be verified against `origin/main`, not against the local working copy.
