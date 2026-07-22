---
name: Commit workflow
description: After completing a task, provide the commit message text only — never execute the commit
type: feedback
originSessionId: 5af14dee-5704-4786-bda3-2b912b2b7349
---
After finishing a task, output only the suggested commit message text. Do not run `git commit` or any git command.

**Why:** The user wants to review and run commits themselves.

**How to apply:** At the end of every task, write the commit message as a code block and stop there. Never call `git commit`, `git add`, or related commands unless explicitly asked.
