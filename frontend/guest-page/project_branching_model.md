---
name: Branching model — feature/staging is the shared test branch, main is production
description: Across all CorePark repos, `feature/staging` is the shared branch wired to CI/CD deploying to the AWS test environment. Anything can be pushed to it (that is the point). `main` is production and cannot be merged into directly — only via PR. Applies to both frontend repos and Java microservices.
type: project
---

## Rule

- **`feature/staging`** — shared test branch. CI/CD auto-deploys to the AWS test environment. **Safe to push anything.** This is where team members converge work for integration testing.
- **`main`** — production. **Direct merges are not allowed / forbidden by policy.** Only PR merges reach main, and the PR must be opened on GitHub AND explicitly authorized (approved) by a reviewer. No self-merges, no CLI merges, no bypassing the GH review flow — even for tiny changes.
- **`develop`** (frontend-guest-page) — analogous role to `feature/staging` in the other repos.

## Why

- Test env exists precisely to absorb in-progress and speculative work before it hits prod. Pushing to staging is the mechanism, not a risk event.
- Prod safety is enforced at the main-branch boundary via PR review + hooks. Anything that skips PR bypasses the human/CI gate.

## How to apply

- **Pushes to `feature/staging` / `develop`**: routine. Don't over-escalate. Divergence, non-fast-forwards, merge commits — all normal on shared integration branches. Reconcile and push. See `feedback_merge_push_divergence.md`.
- **Merges/pushes to `main`**: never do directly, even if the user says "just push it". Path to main is always: open PR on GitHub (`gh pr create`) → wait for reviewer authorization → reviewer merges via GH UI. Do not run `git merge main` locally, do not push to main, do not merge the PR from the CLI. If the user asks me to "merge the PR" or similar, confirm which PR and let them merge via GH themselves unless they explicitly re-authorize the CLI action.
- **Ambiguous target**: if the user says "merge + push" without specifying, ask which branch. Default to the shared staging branch, never main.

## Related

- `feedback_merge_push_divergence.md` — divergence on `feature/staging` is an obstacle, not a scope change; resolve automatically.
- `feedback_commit_workflow.md` — commit workflow default is "message text only, never git commit" — user explicitly overrides per-task when they want direct commits.
