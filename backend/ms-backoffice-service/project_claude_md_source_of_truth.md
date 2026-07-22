---
name: CLAUDE.md is the backend team's shared source of truth
description: The CLAUDE.md at the repo root (main version, dated 2026-06-29) was shared by the backend team as the standard all developments must follow. Not aspirational — it's a contract
type: project
originSessionId: e093e519-e068-4f95-9135-b45cf1bf103f
---
As of 2026-07-06 Israel confirmed that `/Users/israel/Dev/Back-End/ms-backoffice-service/CLAUDE.md` (the version that came from `main`, footer "Last updated: 2026-06-29") is the **backend team's official convention document**. It was handed to Israel as onboarding context when he joined the backend track.

**Why:** it captures the team's agreed patterns — stack, package layout (`bean/` subtree, one slice per feature), DI style (Lombok `@RequiredArgsConstructor`), `@Slf4j` logging, JDBC + `BeanPropertyRowMapper`, and the preferred controller envelope for new code (`ApiResponse<T>` + `ErrorCode` + `GlobalExceptionHandler`). It also flags the legacy pattern (`Response` + `MessageCode` + `RestExceptionHandlerController`) as *don't-introduce-in-new-code*.

**How to apply:**
- Treat CLAUDE.md as normative. When it conflicts with existing code in a slice, that's tech debt to close, not a variant to accept.
- Israel is transitioning from a frontend background into backend and is *learning* these patterns — explain the *why* behind each convention when suggesting refactors, not just the *what*. He hasn't yet built the intuition a long-time Spring dev has.
- The survey admin slice (`com.corepark.backoffice.survey`) shipped 2026-07-02 using the legacy pattern (`Response` + `MessageCode`) and does NOT use the `bean/` subtree — those are the two biggest gaps against CLAUDE.md that Israel wants to close as onboarding work.
