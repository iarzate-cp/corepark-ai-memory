---
name: Avoid the word "blocker/blocked" in team-facing docs — Israel's boss reads it as chaos
description: In shareable docs, status updates, PR descriptions, and any comms that reach stakeholders, replace "blocked/blocker/blocking" with neutral phrasing about what's required or in progress. This is a stakeholder-communication preference, not a code preference.
type: feedback
originSessionId: 6c66ac7f-ac13-4123-8f60-ce19ba5e1720
---
**Rule:** When writing team-facing documents (shareable MDs, status reports, PR descriptions, Jira comments), **do not use the words "blocker", "blocked", "blocking", "block"**. Israel's boss associates that language with chaos and it colors the read of the whole document.

**Why:** 2026-07-09, while writing `coupons-enhancements-status.md` for stakeholders. Israel corrected me: "No le pongas 'bloqueo', a mi jefe no le gusta esa palabra y la relaciona con caos, mejor menciona que es necesario modificar la DB para completar las tareas." It's a communication style thing — the meaning is the same, but framing matters upstream.

**How to apply — replacement phrasing:**

Instead of → use

- "blocked by X" → "requires X first" / "depends on X" / "waiting for X to be in place"
- "This is a blocker" → "This is a dependency" / "This task requires…" / "To complete this, we need…"
- "Blocked on DB access" → "Requires a database update" / "Needs a schema change"
- "What's the block?" → "What needs to change first" / "What this task depends on"
- "This unblocks the flow" → "This enables the flow" / "With this in place, the flow works end-to-end"
- "Blocked tasks" → "Tasks with pending dependencies" / "Tasks that require X first"

**When "blocker" IS OK:**
- Internal conversation between Israel and me (in-session, transient) — he uses "bloqueado" himself in Spanish
- Code comments where technical accuracy matters
- Sprint standup shorthand

**When to definitely avoid it:**
- Files in `~/Documents/*.md` (shareable docs)
- PR descriptions
- Jira ticket comments
- Any doc that will land in front of management

**Meta-rule:** Whenever writing to a stakeholder audience, scan the draft for "block*" words before delivering. If found, rephrase neutrally with dependency/requirement framing.

**Also — stakeholder docs must be plain-language, non-technical:**

2026-07-09, same session. Israel added: "No hagas algo técnico, ni compartas queries, explica en lenguaje natural qué sucede." Stakeholder-facing docs must not include:
- SQL queries or code blocks
- Error codes (like `25006`) or infrastructure names (PgBouncer, information_schema, connection pool)
- Column-type details (VARCHAR, TIMESTAMP WITH TIME ZONE)
- Technical acronyms without explanation
- Anything that reads like an engineer's mental model

Instead, describe in plain English **what the user/business needs to know**:
- "A small update to the way batches are stored" instead of "ALTER TABLE ADD COLUMN"
- "The system needs to record three new details about each batch" instead of listing columns
- "This depends on updating our records format" instead of "requires a schema change"

Think: could a project manager who has never opened a terminal understand this doc without asking follow-up questions? If not, simplify.
