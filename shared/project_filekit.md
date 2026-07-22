---
name: Filekit library project
description: Open-source zero-deps TypeScript library for generating XLSX files and downloading blobs, framework-agnostic
type: project
originSessionId: 4e4fecd7-79f5-4ae6-9a67-dc8722de78c0
---
Location: `/Users/israel/Dev/filekit/` (name is a placeholder — user will rename before publishing).

**Goal:** Open-source, zero-runtime-dependency TypeScript library. Two main capabilities:
1. Generate XLSX files from scratch (writing the ZIP + OOXML ourselves, using STORE mode to avoid a DEFLATE implementation).
2. Trigger blob downloads from the browser (PDF, XLSX, any binary — backend produces the blob, filekit handles the download).

**Non-goals:** wrapping existing libraries, PDF generation (only download), server-side rendering concerns.

**Why:** User believes hand-rolled libraries end up smaller and more efficient than wrappers around ExcelJS/SheetJS/pdf-lib. Bundle target: ~3KB gzip for minimal XLSX, ~6-7KB with styles.

**How to apply:** When working in this repo, follow the strict rules in `CLAUDE.md` (declarative, functional, no ternaries, `.at()` over `[n]`, no third-party runtime deps, only Web Standards APIs). Monorepo layout: `packages/core` (download helper, Result type) and `packages/excel` (ZIP writer + XLSX writer). Stack: pnpm + tsup + Vitest + Biome + Changesets. Alcance de v0.1 explicitly excludes styles, sharedStrings, formulas, merged cells — those come later.
