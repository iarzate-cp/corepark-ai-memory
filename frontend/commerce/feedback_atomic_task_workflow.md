---
name: Work in small, atomic steps — never expand scope silently
description: User strongly prefers task-by-task execution with explicit confirmation between steps. Multi-axis refactors must be enumerated and approved upfront, not bundled. Cross-repo ripple effects must be surfaced before making the change
type: feedback
originSessionId: a2019106-3f08-4f27-8756-0bb019dfebd1
---
Prefer small, atomic tasks executed one at a time with a confirmation checkpoint between each. Do not expand the scope of a task beyond what was explicitly requested, and do not bundle a large refactor under a small ask.

**Why:** On 2026-07-06, during the firebase-cleanup feature, the user asked to bring one backend slice into line with the backoffice CLAUDE.md conventions. Instead of proposing the axes ("Lombok DI, `@Slf4j`, `ApiResponse<T>` migration, OpenAPI docs, tests") and letting the user pick which to attack, I bundled all of them into one pass. The `ApiResponse<T>` migration included a JSON shape change (flat → nested `data`) that forced a matching frontend refactor. The user perceived what they thought was a simple task escalating into hours of work spanning two repos, with steps backward when things broke (Java version, Lombok processor, IntelliJ cache, git commit message mismatches). The task was doable but the way I bundled it burned confidence and disoriented the user. They then asked for this rule to be codified in memory and in the backoffice CLAUDE.md.

**How to apply:**

1. **Enumerate the axes of a broad request before touching code.** When the user asks for a broad instruction ("apply the CLAUDE.md conventions", "align with survey-crud"), list each discrete change I would make. Get explicit approval on which are in scope. Do not "just do it all" because it seems related.

2. **Flag cross-repo ripple effects EXPLICITLY before making the change.** If a backend refactor changes the JSON contract, the frontend has to adapt — that is a cross-repo dependency. Announce it: "This change will require these frontend files to also change. Do you want me to do both, or only the BE?" Never silently expand into a second repo.

3. **Commit after each atomic step, not at the end.** Small commits keep the working tree small and let the user roll back a single change without losing the whole batch. If working tree accumulates 5 unrelated tweaks, the user has lost the ability to reason about them one at a time.

4. **Match the commit's message to its actual diff.** If a chore commit says "align env config with survey-crud convention", the diff must also actually apply that alignment (e.g., update `environment.ts`). Do not leave a message that overstates the diff. Verify diff ↔ message before committing.

5. **When in doubt, ask.** "Should I also do Y?" is one sentence. Doing Y silently and having to unwind it later is many turns of frustration.

6. **When the user delegates a larger scope explicitly** ("hazlo todo hasta el push"), the atomic rule still applies internally: enumerate the sub-steps in the initial reply, do them in order, report each result, and stop at the delegated boundary.

Also captured in the ms-backoffice-service CLAUDE.md under `## Local iteration — atomic changes`, scoped there as a local-development rule for that repo. The memory version is the general rule for how to work with this user across all repos.
