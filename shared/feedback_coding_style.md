---
name: Coding style preferences (functional, declarative, no ternaries)
description: Durable coding style preferences the user applies to his own projects — functional/declarative, no ternaries, .at() over numeric indexing, early returns
type: feedback
originSessionId: 4e4fecd7-79f5-4ae6-9a67-dc8722de78c0
---
For the user's own projects (personal libraries, side projects), enforce these style rules by default. He has stated them as non-negotiable.

**Rules:**
- **Declarative, never imperative.** Use `.map`/`.filter`/`.reduce`/`.flatMap` over `for`/`while` loops. No accumulator mutation.
- **Functional programming everywhere.** Pure functions, immutability, composition. Side effects only at I/O boundaries.
- **Early returns always.** No unnecessary nesting.
- **No ternary operators.** Substitute with `if/else` + early return, object lookup by key, named helper functions, or `switch(true)`. He finds ternaries hard to read and wants the code accessible to collaborators.
- **Never numeric bracket indexing.** Use `arr.at(0)` / `arr.at(-1)`, never `arr[0]` or `arr[i]`.
- **Modern JS/TS features**: `satisfies`, `const` type params, template literal types, branded types, discriminated unions.
- **No classes for stateless logic.** Factory functions.
- **No barrel files inside packages**, only at package entry.

**Why:** He believes these rules produce more readable, maintainable, and efficient code, and wants any collaborator on his open-source projects to follow the same conventions from day one.

**How to apply:** When writing or reviewing code in his personal projects (anything under `/Users/israel/Dev/` that he owns), enforce all of the above without asking. When working on other people's codebases or client work, follow the existing conventions of that codebase instead.
