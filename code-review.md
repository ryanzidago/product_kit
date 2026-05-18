# Code Review

Use code review to identify concrete bugs, regressions, missing safeguards, and test gaps. Lead with findings, ordered by severity, and ground each finding in specific files, lines, behavior, or failing scenarios.

For each material finding:

- State the concrete issue first.
- Explain why it matters.
- Suggest a direct fix.
- Call out uncertainty explicitly when the evidence is incomplete.

## Proving Findings With Regression Tests

When asked to prove code review findings, draft focused failing regression tests for each material finding before proposing or implementing the fix.

Each regression test should include an `@doc` immediately above the test explaining:

- `Issue:` what the bug or regression is.
- `Why it matters:` the user, data, operational, security, or maintenance impact.

Keep the test body focused on reproducing the reviewed issue. Do not use the test to encode speculative cleanup or unrelated behavior.

## PR Comment Format For Proven Findings

When posting a proven finding to a PR, prefer an inline comment on the relevant changed line. Keep the comment focused on the evidence.

Use this format:

1. One short sentence naming the issue.
2. A fenced code block containing the full failing regression test, including its `@doc`.

Do not add a long prose explanation outside the test unless the risk cannot be understood from the `@doc` and test name. The `@doc` is the main explanation of the issue and impact.
