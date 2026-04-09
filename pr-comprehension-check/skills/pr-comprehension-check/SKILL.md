---
name: pr-comprehension-check
description: Interactive multiple-choice quiz on a diff or PR to verify the developer understands the changes before merging.
compatibility:
  tools:
    - AskUserQuestion
    - Bash
    - Grep
    - Glob
    - Read
---

# PR Comprehension Check

## Usage

```
/pr-comprehension-check              # Quiz on current branch vs main
/pr-comprehension-check 42           # Quiz on PR #42
/pr-comprehension-check <pr-url>     # Quiz on a specific PR URL
```

- **No arguments**: diffs the current branch against the base branch (`main`, `master`, or `dev`)
- **PR number or URL**: fetches the diff and metadata via `gh pr view` / `gh pr diff`

## Workflow

### Step 1: Gather Context

#### Mode A: PR number or URL provided

```bash
gh pr view <number-or-url> --json title,body,baseRefName,headRefName,files,additions,deletions
gh pr diff <number-or-url>
gh pr diff <number-or-url> --stat 2>/dev/null || gh pr view <number-or-url> --json files
gh pr view <number-or-url> --json commits --jq '.commits[].messageHeadline'
```

#### Mode B: No arguments (current branch)

Try these diff commands in order until one works:

```bash
git diff main...HEAD          # against main/master/dev
git diff origin/main...HEAD   # against remote
git diff --cached             # staged only
git diff                      # all uncommitted
```

```bash
git diff main...HEAD --stat
git log main..HEAD --oneline
```

Use `Grep` and `Glob` to find files that import or depend on changed modules — this grounds blast-radius questions in reality.

### Step 2: Analyze the Diff

Before generating questions, build a mental model:

1. **Structural changes** — new/deleted/renamed files, modified interfaces
2. **Architectural decisions** — patterns chosen, abstractions introduced, dependencies added
3. **Alternatives not taken** — what other approaches could have solved this
4. **Blast radius** — affected modules, services, APIs, routes (verify with Grep)
5. **Deployment surface** — containers, services, environments, config touched
6. **Risk areas** — race conditions, breaking changes, migrations, security implications

### Step 3: Determine Question Count

| Change size | Criteria | Questions |
|---|---|---|
| Small | <100 lines, 1-2 files | 3 |
| Medium | 100-500 lines, or 3-8 files | 4 |
| Large | 500+ lines, or 9+ files, or architectural changes | 5 |

Config-only or test-only changes: 3 questions.

### Step 4: Generate and Ask Questions

Generate all questions up front, then present them **one at a time** using `AskUserQuestion`. Each question has 3-4 options — one correct, the rest plausible but wrong. Randomize the position of the correct answer.

#### Question Categories

Pick from these based on relevance to the diff. Each question should reference specific file names, function names, or module names from the changes — not generic concepts.

- **Intent & Architecture** (at least 1): Why this approach over alternatives. What pattern was introduced and why.
- **Blast Radius** (at least 1): What breaks if something goes wrong — which modules, endpoints, flows are affected.
- **Deployment & Infrastructure** (if relevant): Migration concerns, config changes, deployment order.
- **Risk & Edge Cases** (at least 1): Race conditions, backwards compatibility, failure modes, rollback plan.
- **Verification** (if room): What test or check would catch a regression.

#### Decoy Question (exactly 1 per quiz)

Include one trick question that presents a real code snippet from the repo that was **NOT changed** in the PR, framed as if it were part of the change. The correct answer is always a variant of "This code was not modified in this PR."

Pick code that is thematically related but untouched — same directory or similar domain. Use the `preview` field on `AskUserQuestion` options to show the snippet. Include 2-3 plausible reasons the code *could* have changed alongside the correct "not modified" option. The decoy catches developers who are guessing without having read the diff.

The decoy counts toward the total question count. Place it at a random position — not always last.

#### Difficulty Mix

- 1 Basic (skimming the diff is enough)
- 1-2 Intermediate (requires understanding the architecture)
- 1-2 Advanced (requires understanding tradeoffs and failure modes)

Prefix each question with number and difficulty: "Q1/4 [Basic] — ..."

After each answer, briefly say if they got it right or wrong (one-line explanation for wrong answers), then immediately ask the next question. Keep momentum.

### Step 5: Score and Assess

Present a summary:

```
## Comprehension Check Results

Score: X/Y (percentage%)

Q1 [Architecture] ... correct/incorrect
Q2 [Blast Radius] ... correct/incorrect
...
```

Readiness assessment:

- **Ready to ship** (all correct, or 1 wrong on Basic): Proceed with PR creation.
- **Review recommended** (1-2 wrong on Intermediate/Advanced): Explain missed concepts, offer to proceed or re-quiz on missed areas.
- **Not ready** (3+ wrong, or wrong on both Blast Radius and Risk): Walk through the changes at an architectural level. Offer a re-quiz with new options on missed questions.

## Example

**Q1/3 [Basic] — Architecture:** "This PR switches session handling from JWT in localStorage to httpOnly cookies. What is the primary security benefit?"
- A) Prevents XSS attacks from accessing tokens
- B) Reduces server memory usage
- C) Enables faster token validation

Developer picks A. "Correct!"

**Q2/3 [Intermediate] — Decoy:** "The following function was also updated in this PR. What was the purpose of the change?"
*(Shows preview of `refresh_token()` from `auth/tokens.py` — NOT modified in the PR)*
- A) Added retry logic for expired refresh tokens
- B) This code was not modified in this PR
- C) Switched from symmetric to asymmetric signing

Developer picks B. "Correct! This function wasn't touched — good eye."
