---
name: SonarCloud uses a "New Code" baseline — touching a legacy line surfaces its issues
description: Sonar's Quality Gate only scrutinizes lines added/modified in the PR. Grandfathered legacy code with the same issue does not block. Improving a legacy line can therefore turn a green branch red. Playbook for handling this
type: feedback
originSessionId: 389cf61a-37a2-48a8-b1d5-8fd8e7808e3c
---
Before modifying any line of pre-existing code in ms-backoffice-service, remember that SonarCloud's Quality Gate uses a "New Code" baseline: **only lines added or modified in the branch are analyzed strictly**. Legacy lines with the same problem are grandfathered — they show up as issues in Overall Code but do not block the gate. This has a counter-intuitive consequence: **a technically-correct improvement to a legacy line can turn a green branch red**, because the touched line becomes new-code and its pre-existing rule violation is now blocking.

**Why:** on 2026-07-08, on PR #85 (`feature/survey-admin-crud`), I tightened two verbose `LOGGER.info` calls in `SurveyServiceImpl` — the old versions dumped the entire `SurveyQuestionnaire` / `PetitionQuestionnaireSubmitRequest` trees at INFO level; my versions logged just the UUID. Objectively safer + less noisy. But Sonar's rule S5145 ("log user-controlled data") flags any request-derived value in a log call, and it now applied to my "new-code" lines even though the OLD versions had the exact same issue and were grandfathered. The PR flipped from green (before commit `3698d36`) to red (after). Fixing it required four extra commits (`cb9a646` constants extraction — resolved a different unrelated rule cascade, `0e61c2c` UUID extraction — made it worse by creating more "new" lines, `6924229` @SuppressWarnings — did NOT take effect, `e352748` inline `// NOSONAR` — finally worked).

Additionally on the same PR, the constants-extraction refactor in `cb9a646` unblocked a different set of red X's ("Define a constant instead of duplicating this literal N times" in `SurveyAdminDaoImpl`). Those issues existed in the file since it was created by an earlier commit on the same branch (`f381453`), but they only surfaced against MY commit — either because Sonar's rules had been tightened between runs, or because the baseline moved.

**How to apply:**

- Before touching any pre-existing line in this repo, do a mental pass: what Sonar rules could apply to the new form of the line? For log statements, string literal handling, exception handling, method complexity, and try/catch bodies especially.
- Prefer commits that only add new files (they get the full new-code treatment, but nothing was previously grandfathered — so nothing surprises you). Refactor commits that modify legacy lines require more caution.
- If the commit is unavoidable and Sonar turns red on rules like S5145 that apply to typed-safe values (UUIDs, enums, IDs), the escalation ladder is:
  1. **First**: try `@SuppressWarnings("java:SXXXX")` at method level with a comment explaining why. Cheap, standard Java. Works for most Sonar rules.
  2. **If step 1 fails** (as it did for S5145): use `// NOSONAR — <one-line reason>` at the end of the offending line. This is Sonar's universal "emergency brake" and works even when `@SuppressWarnings` doesn't propagate.
  3. **If step 2 also fails** (rule is classified as a Security Hotspot in this Sonar config): go to the SonarCloud dashboard, open the hotspot, click "Change Status" → "Safe" with a comment. Cannot be resolved in code.
- The reason string on the `NOSONAR` comment matters — future reviewers see it. Format: `// NOSONAR — <why the rule doesn't apply here>`. E.g. `// NOSONAR — java.util.UUID.toString() is fixed-format hex/dashes, no injection surface`.
- Intermediate commits on the branch keep their red X's forever — Sonar analyzes each commit individually and does not backfill. Only the HEAD commit matters for merge. Don't panic when a commit deep in the history looks failed; check the tip.
- Similar lesson applies to any static-analysis gate on a PR baseline: not just Sonar — Codecov, SpotBugs, etc. Ask the same question before modifying legacy code.
- When explaining this to the team, the framing that lands well is *"my improvement was flagged because Sonar doesn't compare before-vs-after — it only asks whether the line was touched"*. This is the mental model to bring to reviewers who ask "but you made it more secure, why is Sonar complaining?".
