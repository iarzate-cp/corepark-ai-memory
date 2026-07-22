---
name: Shareable docs go to ~/Documents/, not the repo
description: When Israel asks for docs meant for team-sharing, generate to ~/Documents with a dated slug. Offer HTML for visual polish + MD for the technical mirror
type: feedback
originSessionId: e41472d4-2583-420e-b7e9-9700af16d6ff
---
Shareable documents (engineering reports, QA plans, user guides) go to `~/Documents/`, not into any repo.

**Why:** Israel explicitly on 2026-07-02 asked to save docs there — "crea un archivo MD en Documents" and later "un HTML que podamos compartir". Docs are consumed by the team via Slack/email/direct file share; they do not live in git and shouldn't clutter branches.

**How to apply:**

- When Israel asks for a document meant to be shared with a person or team, default to `~/Documents/<kebab-slug>-YYYY-MM-DD.<ext>`.
- Match the format to the audience:
  - **Technical (dev / QA engineer):** MD works well. Offer an HTML companion if visual polish matters or if it will be shared cross-org.
  - **Non-technical (Customer Service, operators, business):** HTML with plain language, larger type, UI-element callouts, FAQ accordions. Language: Spanish by default for internal non-tech audiences at Corepark.
- HTML should be **self-contained**: all CSS/JS inline, Google Fonts via CDN link is OK (no local font hosting needed). Under 100 KB total keeps it Slack-friendly.
- Non-technical HTML: avoid `<code>` for endpoints or configuration; use `<span class="ui">Button Label</span>` for UI elements the user will interact with.
- Include a bottom "how to report a problem" section pointing to Israel by email/Slack.

**Filename convention:** `<repo-or-topic>-<kind>-<date>.<ext>`. Examples from 2026-07-02:
- `ms-backoffice-survey-testing-2026-07-02.md`
- `ms-backoffice-survey-testing-2026-07-02.html`
- `frontend-commerce-survey-qa-2026-07-02.html` (technical, English)
- `guia-encuestas-corepark-2026-07-02.html` (non-technical, Spanish)

**When repo docs might make more sense:** if the content is a testing convention or codebase reference that other developers will need while working in the repo, an MD in the repo (e.g., `docs/testing.md`) may be better. Confirm before writing to the repo.
