---
name: Prefers building from scratch over wrapping dependencies
description: When building his own libraries, user strongly prefers zero third-party dependencies and from-scratch implementations
type: feedback
originSessionId: 4e4fecd7-79f5-4ae6-9a67-dc8722de78c0
---
For his own libraries/projects, the user prefers writing everything from scratch with zero third-party dependencies.

**Why:** He has found that his hand-rolled libraries end up more efficient and cover use cases that existing libraries don't. It's a deliberate quality/control preference, not NIH syndrome.

**How to apply:** When he's building his own tooling or libraries, don't reflexively suggest wrapping `pdf-lib`, `exceljs`, `lodash`, etc. Default to explaining the underlying format/spec and building minimal implementations. Web Standards APIs (CompressionStream, TextEncoder, fetch, Web Streams) are fine — those count as platform, not dependencies. This does *not* apply to work on other people's projects where he's just consuming existing stacks.
