# Communication Guide

## Presenting Options

When presenting a list of options, always number them so the user can refer to choices by number.

**Do:**

```
1. **Option A** — description
2. **Option B** — description
3. **Option C** — description
```

**Don't:**

```
- **Option A** — description
- **Option B** — description
- **Option C** — description
```

## Enumerating Findings

For review feedback, use labeled identifiers:

- Findings: `F1`, `F2`, `F3`, ...
- Recommendations: `R1`, `R2`, `R3`, ...

Keep each item self-contained. When a recommendation addresses a finding, reference it directly, for example `R2 addresses F1`. In follow-up replies, refer to items by label.

Example:

```md
Findings

- F1. Missing provider correlation ID in `/path/to/project/.worktrees/tool-execution/.artefacts/2026-04-04-tool-execution-v1/design.md:78`.
- F2. Retry semantics are not idempotent in `/path/to/project/.worktrees/tool-execution/.artefacts/2026-04-04-tool-execution-v1/design.md:323`.

Recommendations

- R1. Add `provider_tool_call_id` and link the tool call to the assistant message that emitted it. Addresses F1.
- R2. Define claim-and-execute semantics so retries cannot double-run a tool. Addresses F2.
```

For code review style responses, use this structure:

1. Findings
2. Open questions or assumptions
3. Recommendations

## Verify Assumptions

Do not make assumptions — verify them. Before stating something as fact or basing a decision on an assumption, confirm it using available tools:

1. **Search the code** — Grep, Glob, Read
2. **Query the database** — DB MCP tools
3. **Search the web / read official docs** — WebSearch, WebFetch
4. **Write a script** — Bash to test a hypothesis
5. **Ask the user** — when none of the above can resolve it

If verification fails after **3 attempts**, stop and ask the user rather than proceeding on an unverified assumption.

## Public References

When mentioning a public GitHub issue, pull request, discussion, commit, release, or other public documentation page in user-facing prose, use an inline Markdown link to the canonical URL at first mention.

Do not leave public references as bare `#123`, `owner/repo#123`, raw URLs, or unlinked titles when a canonical URL is available.

Do not create a separate links-only section unless the task explicitly asks for one.

**Do:**

```md
See [#126](https://github.com/example/project/pull/126).
See the [streaming design plan](https://github.com/example/project/blob/main/.artefacts/example.md).
```

**Don't:**

```md
See #126.
See the streaming design plan.

Links:
- https://github.com/example/project/pull/126
```

## File References

When mentioning files, always use the full absolute path (e.g. `/path/to/project/lib/project/accounts.ex`) rather than a relative or abbreviated path. This makes references work consistently regardless of context — for example when working in git worktrees or when opening files from any location.
