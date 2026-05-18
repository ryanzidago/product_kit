# Retrospective

Run a structured retrospective on a merged PR. Analyze what went wrong, what could be improved, and which automatable guards would have prevented the problems.

## Goal

Produce a thorough, actionable retrospective for a merged PR. The output is a markdown document saved to the appropriate feature folder in `artefacts/`. The retrospective focuses on **problems**, **improvements**, and **automatable guards**. Do not spend space celebrating what went well.

## Prerequisites

- `gh` CLI authenticated with access to the repo
- Inside a git repository (or a subdirectory of one)
- The PR is merged, or at minimum the diff is available

## Input

The user provides one of:

- A PR number or URL
- A branch name
- Nothing, in which case use the current branch's most recent PR

## Process

### Phase 1: Gather Evidence

#### Step 1: Resolve the PR

If the user provided a PR number or URL, extract the number. Otherwise resolve it from the current branch:

```bash
gh pr view --json number -q .number
```

#### Step 2: Fetch PR metadata

```bash
gh pr view <number> --json title,body,baseRefName,headRefName,url,additions,deletions,changedFiles,commits,createdAt,mergedAt,reviews,comments
```

Capture:

- PR URL, title, branch names
- Size metrics: files changed, lines added and removed, commit count
- Timeline: created to merged duration
- Review comments from humans and bots

#### Step 3: Fetch the full diff

```bash
gh pr diff <number>
gh pr diff <number> --name-only
```

#### Step 4: Fetch review comments

```bash
gh api repos/{owner}/{repo}/pulls/<number>/comments --paginate
gh api repos/{owner}/{repo}/pulls/<number>/reviews --paginate
gh api repos/{owner}/{repo}/issues/<number>/comments --paginate
```

These contain the actual review findings. Parse them for substance.

#### Step 5: Read changed files in full

For each changed file, skipping deleted files, read the full file at `HEAD` to understand the final state and surrounding context.

#### Step 6: Check CI results

```bash
gh pr checks <number>
```

Note any failures, flaky tests, or warnings that occurred during the PR lifecycle.

### Phase 2: Analyze

For each finding, cite specific files, line numbers, commit SHAs, or review comments as evidence.

#### Section A: What Went Wrong

Identify concrete problems that actually occurred during the PR lifecycle.

Use these severities:

| Severity | Meaning |
|----------|---------|
| **Critical** | Bug shipped, security issue, data integrity risk, or production impact |
| **Major** | Significant logic error caught in review, spec and implementation divergence, missing auth check |
| **Minor** | Style violations, copy-paste errors, or test gaps caught before merge |

Look for:

- Bugs that shipped
- Bugs caught late
- Spec and implementation divergence
- Security issues
- Copy-paste errors
- Convention violations
- Test gaps
- Noisy commit history

#### Section B: What Could Be Improved

Identify forward-looking process and practice improvements.

For each improvement, be explicit about:

1. What to change
2. Why it matters, linked back to a "What Went Wrong" finding
3. When it applies

Common categories:

- Design process
- Code review
- Testing strategy
- Commit hygiene
- Knowledge gaps

#### Section C: Automatable Guards

For every finding in sections A and B, ask whether it can be made impossible or automatically detectable.

For each proposed guard, include:

1. What it catches
2. The tool to use
3. An implementation sketch
4. False-positive risk and how to minimize it

Evaluate these automation categories:

- Tests (ExUnit)
- Credo checks
- ast-grep rules
- CI checks
- Mix tasks
- Pre-commit or pre-push hooks
- Compiler warnings

Rate each guard:

| Rating | Meaning |
|--------|---------|
| **Quick win** | Less than 1 hour to implement, prevents a real class of bugs |
| **Worth it** | 1 to 4 hours, prevents a significant recurring issue |
| **Consider** | More than 4 hours or high false-positive risk, but addresses a real problem |
| **Skip** | Too much effort relative to the risk, or the problem is unlikely to recur |

### Phase 3: Write the Report

#### Step 7: Determine the output location

The retrospective goes in the feature's artefacts folder.

1. Check whether a feature folder already exists in `.artefacts/` for this work
2. If yes, save there as `.artefacts/{date-slug}/YYYY-MM-DD-retrospective.md`
3. If no, create a new feature folder based on the PR branch name

Use today's date for the retrospective file timestamp.

#### Step 8: Write the retrospective document

Use this structure:

```markdown
# Retrospective: PR #<number> — <title>

**PR**: <url>
**Branch**: `<head>` -> `<base>`
**Timeline**: <created> to <merged> (<duration>)
**Size**: <files> files changed, +<additions> / -<deletions> lines, <commits> commits

---

## Key Metrics

| Metric | Value |
|--------|-------|
| Time to merge | ... |
| Commits | ... |
| Files changed | ... |
| Lines added/removed | ... |
| New tests | ... |
| CI result | ... |
| Review findings (human) | ... |
| Review findings (bot) | ... |

---

## What Went Wrong

### 1. <Title> [Critical/Major/Minor]

<Description with specific file:line references, commit SHAs, and review comment links>

**Evidence**: <quote from review comment, diff snippet, or test output>

---

## What Could Be Improved

### 1. <Title>

<What to change, why, and when it applies>

**Addresses**: "What Went Wrong" #<N>

---

## Automatable Guards

### 1. <Guard title> [Quick win / Worth it / Consider / Skip]

**Catches**: "What Went Wrong" #<N>
**Tool**: <ExUnit / Credo / ast-grep / CI / Mix task / Hook / Compiler>
**Implementation**: <Concrete implementation sketch>
**False positive risk**: <Assessment>
```

## Standards

- Be honest, not kind. The point is to find problems.
- Do not invent hypotheticals and present them as facts.
- Tie each finding to concrete evidence from the PR lifecycle.
- Prefer a smaller number of sharp findings over a long list of weak ones.
