---
name: Shareable docs go to ~/Documents/, not the repo
description: When Israel asks for docs to share with the team, save to ~/Documents with a dated slug. Offer HTML for visual polish + MD when useful for the technical mirror
type: feedback
---

Shareable documents (engineering reports, QA plans, user guides) go to `~/Documents/`, not into any repo.

**Why:** Israel explicitly on 2026-07-02 asked to save docs there — "crea un archivo MD en Documents" and later "un HTML que podamos compartir". Docs are consumed by the team via Slack/email/direct file share; they don't live in git.

**How to apply:**

- When Israel asks for a document meant to be shared, default to `~/Documents/<kebab-slug>-YYYY-MM-DD.<ext>`.
- Match the format to the audience:
  - **Technical (dev / QA engineer):** MD works well. Offer an HTML companion if visual polish matters.
  - **Non-technical (Customer Service, operators, business):** HTML with plain language, larger type, UI-element callouts. Language: Spanish by default for internal Corepark non-tech audiences.
- HTML should be **self-contained**: all CSS/JS inline; Google Fonts via CDN is OK. Under 100 KB is Slack-friendly.
- Non-technical HTML: avoid `<code>` for endpoints or config; use `<span class="ui">Button Label</span>` for UI elements the user will interact with.
- Include a bottom "how to report a problem" section with Israel's contact.

**Filename convention:** `<repo-or-topic>-<kind>-<date>.<ext>`. Examples from 2026-07-02:
- `ms-backoffice-survey-testing-2026-07-02.md`
- `ms-backoffice-survey-testing-2026-07-02.html`
- `frontend-commerce-survey-qa-2026-07-02.html`
- `guia-encuestas-corepark-2026-07-02.html`
