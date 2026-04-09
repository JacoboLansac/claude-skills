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

## Purpose

When an LLM writes or substantially modifies code on behalf of a human, the human must still own the changes.
This skill runs an interactive multiple-choice quiz — one question at a time — that tests whether the developer
genuinely understands what changed, why, and what could go wrong. The developer picks answers from options
instead of typing, making it fast to complete while still being a meaningful gate.

## Usage

```
/pr-comprehension-check              # Quiz on current branch vs main
/pr-comprehension-check 42           # Quiz on PR #42
/pr-comprehension-check <pr-url>     # Quiz on a specific PR URL
```

- **No arguments**: diffs the current branch against the base branch (`main`, `master`, or `dev`)
- **PR number**: fetches the diff and metadata from that PR via `gh pr view` / `gh pr diff`
- **PR URL**: same as PR number, but accepts a full GitHub PR URL (e.g. `https://github.com/owner/repo/pull/42`)

## When to Run

- At the end of a Claude Code programming session, before `gh pr create` or equivalent
- As a gate in any PR creation workflow
- On demand when the developer wants to verify their understanding of a diff

## Workflow

### Step 1: Gather Context

Collect the diff and any available PR/issue context. The method depends on how the skill was invoked.

#### Mode A: PR number or URL provided

If the user passed a PR number or URL, use `gh` to fetch everything:

```bash
# Get PR metadata (title, body, base branch, linked issues)
gh pr view <number-or-url> --json title,body,baseRefName,headRefName,files,additions,deletions

# Get the full diff
gh pr diff <number-or-url>

# Get the diff summary
gh pr diff <number-or-url> --stat 2>/dev/null || gh pr view <number-or-url> --json files

# Get commit history
gh pr view <number-or-url> --json commits --jq '.commits[].messageHeadline'
```

#### Mode B: No arguments (current branch)

Diff the current branch against the base branch:

```bash
# Get the diff — try these in order:
# Option A: Diff of current branch against the main branch (main, master, or dev)
git diff main...HEAD

# Option B: If that fails, try against origin/main
git diff origin/main...HEAD

# Option C: Staged changes only
git diff --cached

# Option D: All uncommitted changes
git diff
```

```bash
# Get the diff summary (always run this for sizing)
git diff main...HEAD --stat

# Get commit history on this branch
git log main..HEAD --oneline
```

**Get linked issue context via `gh` CLI** (if available):
```bash
# Check for linked issues from branch name or recent commits
gh issue view <number> 2>/dev/null
# Or list recent issues for context
gh issue list --limit 5 --state open
```

**Search the codebase for blast radius** — use `Grep` and `Glob` to find files that import or depend
on changed modules. This makes blast-radius questions grounded in reality rather than hypothetical.

### Step 2: Analyze the Diff

Before generating questions, build a mental model:

1. **What changed structurally** — new files, deleted files, renamed files, modified interfaces
2. **Architectural decisions** — patterns chosen, abstractions introduced, dependencies added
3. **Alternatives not taken** — what other approaches could have solved this
4. **Blast radius** — which modules, services, APIs, routes, or components are affected (use Grep to verify)
5. **Deployment surface** — containers, services, environments, config, infrastructure touched
6. **Risk areas** — race conditions, breaking changes, migration needs, security implications

### Step 3: Determine Question Count

Scale the number of questions to the complexity of the change:

| Change size | Lines changed | Question count |
|---|---|---|
| Small | <100 lines, 1-2 files | 3 questions |
| Medium | 100-500 lines, or 3-8 files | 4 questions |
| Large | 500+ lines, or 9+ files, or architectural changes | 5 questions |

Config-only or test-only changes: 3 questions, focused on their specific domain.

### Step 4: Generate and Ask Questions Iteratively

This is the core of the skill. Generate all questions up front, then present them **one at a time** using
`AskUserQuestion`. Each question has 3-4 answer options — one correct, the rest plausible but wrong.

#### Question Categories

Pick questions from these categories based on what's relevant to the diff:

**Intent & Architecture** (include at least 1):
Why the approach was chosen over alternatives. What pattern or abstraction was introduced and why.

**Blast Radius & Affected Paths** (include at least 1):
What is affected by the changes — which modules, endpoints, flows break if something goes wrong.

**Deployment & Infrastructure** (include if relevant):
What needs redeploying, migration concerns, config changes, environment variables.

**Risk & Edge Cases** (include at least 1):
What could go wrong — race conditions, backwards compatibility, failure modes, rollback plan.

**Verification** (include if room):
What test or check would catch a regression in the changed area.

**Decoy / Red Herring** (include exactly 1 per quiz):
This is a trick question. Present a real code snippet from the repository that was **NOT changed or affected
by the PR**, and frame a question as if it were part of the change. The correct answer is always some variant
of "This code is not part of this PR" or "This is unrelated to the changes." This tests whether the developer
actually knows the boundaries of what they changed versus what was already there.

How to build a good decoy:
1. Use `Grep` or `Read` to find a real code snippet from a file near the changed files — same directory,
   similar domain, or a file that imports/is imported by the changed code. Proximity makes it more believable.
2. Pick code that is thematically related but untouched. For example, if the PR fixes auth token parsing,
   show a nearby function that handles token refresh and ask about it as if it changed.
3. Use the `preview` field on AskUserQuestion options to show the actual code snippet, so the developer
   has to visually inspect it.
4. Frame the question naturally — "This function was also modified in the PR. What was the reason for
   the change?" — so it doesn't scream "trick question."
5. Among the options, include 2-3 plausible-sounding reasons the code *could* have been changed, plus
   one option like "This code was not modified in this PR." Place the correct "not modified" answer at
   a random position.

The decoy question serves a critical purpose: it catches developers who are just guessing or who approved
changes without actually reading the diff. Someone who truly reviewed the code will immediately recognize
that a snippet wasn't part of their change.

#### How to Ask Each Question

Use `AskUserQuestion` with a single question per call. Present the question with a difficulty tag in the
header field, and provide 3-4 options where exactly one is correct.

**Difficulty distribution across the quiz:**
- 1 Basic question (anyone who skimmed the diff could answer)
- 1-2 Intermediate questions (requires understanding the architecture)
- 1-2 Advanced questions (requires understanding tradeoffs and failure modes)
- 1 Decoy question (tests whether the developer knows the boundaries of their change)

The decoy counts toward the total question count (3-5). For a 3-question quiz, use 1 real + 1 real + 1 decoy.
Place the decoy at a random position in the quiz — not always last.

**Example of a single question call:**

Use `AskUserQuestion` with:
- `header`: Short tag like "Architecture" or "Blast Radius" (max 12 chars)
- `question`: The full question text, prefixed with the question number and difficulty emoji (e.g., "Q1/4 - What security vulnerability does switching from localStorage JWT to httpOnly cookies address?")
- `options`: 3-4 choices, with the correct answer placed at a **random position** (not always first!)
- `multiSelect`: false

Difficulty emojis to include in the question text:
- "Basic" for straightforward questions
- "Intermediate" for architecture-level questions  
- "Advanced" for tradeoff/failure-mode questions

**Important: randomize the position of the correct answer across questions.** Do not always put it first or last.
Make the wrong options genuinely plausible — they should represent real alternatives or common misconceptions,
not obvious throwaways.

After the developer selects an answer, briefly tell them if they got it right or wrong, with a one-line
explanation for wrong answers. Then immediately ask the next question. Keep momentum — this should feel
like a quick quiz, not a lecture.

### Step 5: Score and Assess

After all questions are answered, present a summary:

```
## Comprehension Check Results

Score: X/Y (percentage%)

Q1 [Architecture] ... correct/incorrect
Q2 [Blast Radius] ... correct/incorrect
...
```

Then give a readiness assessment:

- **Ready to ship** (all correct, or 1 wrong on a Basic question): Proceed with PR creation.
  Note in the PR description that the author passed a comprehension check.
- **Review recommended** (1-2 wrong on Intermediate/Advanced): Briefly explain the missed concepts,
  then offer to proceed or re-quiz on the missed areas.
- **Not ready** (3+ wrong, or wrong on Blast Radius + Risk): Walk through the changes together
  at an architectural level. Offer a second attempt at the missed questions before proceeding.

### Step 6: Proceed or Loop

- If **Ready to ship**: Tell the developer they're good to go. If a PR creation workflow follows,
  proceed with it.
- If **Review recommended**: Use `AskUserQuestion` to ask if they want to proceed anyway, get a
  walkthrough of missed areas, or re-take the missed questions.
- If **Not ready**: Explain the key areas of concern. Offer a targeted re-quiz on just the missed
  questions (generate new options so they can't just memorize the answer).

## Writing Good Questions and Options

The quality of the quiz depends entirely on the quality of the distractors (wrong options). Here's how
to make them work:

- **Distractors should be real alternatives.** If the question is about why pattern X was chosen,
  the wrong options should be other patterns that a reasonable developer might consider — not absurd
  strawmen.
- **Avoid "all of the above" or "none of the above."** These are lazy and don't test understanding.
- **Ground questions in the actual diff.** Reference specific file names, function names, module names
  from the changes. Generic questions like "what's the rollback plan?" are weak; "if the migration in
  `003_add_session_tokens.sql` fails halfway, what state is the database left in?" is strong.
- **Use blast-radius search results.** If you used Grep to find that 4 files import a changed module,
  you can ask "how many modules directly depend on the changed interface?" with concrete number options.

## Example Flow

For a medium-complexity auth change (5 questions):

**Q1/5 - Basic [Architecture]:** "This PR switches session handling from JWT in localStorage to
httpOnly cookies. What is the primary security benefit?"
- A) Prevents XSS attacks from accessing tokens
- B) Reduces server memory usage
- C) Enables faster token validation
- D) Allows sessions to persist across browsers

Developer picks A. "Correct!"

**Q2/5 - Intermediate [Blast Radius]:** "Which of these modules is NOT directly affected by
the auth middleware change?"
- A) `/api/users` routes
- B) `/api/payments` routes
- C) The static file server
- D) `/api/admin` routes

Developer picks B. "Not quite — the payments routes use authenticated endpoints too. The static
file server (C) is the one that doesn't go through auth middleware."

**Q3/5 - Advanced [Risk]:** "During deployment, there will be a window where old JWTs and new
cookies coexist. What happens to users with active sessions?"
- A) They're automatically migrated to cookies on next request
- B) They'll be logged out and need to re-authenticate
- C) The old JWT middleware runs as fallback for 24 hours
- D) Sessions are preserved via a dual-read strategy in the new middleware

Developer picks D. "Correct!"

**Q4/5 - Intermediate [Decoy]:** "The following function was also updated in this PR.
What was the purpose of the change?"
*(Shows a code preview of `refresh_token()` from `auth/tokens.py` — a real function
that was NOT modified in the PR)*
- A) Added retry logic for expired refresh tokens
- B) Changed the token expiry from 24h to 1h
- C) This code was not modified in this PR
- D) Switched from symmetric to asymmetric signing

Developer picks C. "Correct! This function wasn't touched — good eye."

**Q5/5 - Intermediate [Deployment]:** "What is the correct deployment order for this change?"
- A) Database migration, then auth service, then API gateway
- B) API gateway, then auth service, then database migration
- C) Auth service only — no other services need redeployment
- D) All services simultaneously via blue-green deployment

Developer picks A. "Correct!"

**Result: 4/5 — Review recommended.** One miss on blast radius. Brief explanation provided,
developer confirms they understand and proceeds to PR creation.
