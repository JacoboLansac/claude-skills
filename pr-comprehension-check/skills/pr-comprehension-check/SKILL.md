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
| Large | 500+ lines, or 9+ files, or architectural changes | 4 |

Config-only or test-only changes: 3 questions.

**Maximum 4 questions** — this is a hard limit imposed by `AskUserQuestion`.

### Step 4: Generate and Ask Questions

**CRITICAL — No narration.** Do not output any text between finishing the analysis and asking the first question. Never mention "preparing questions", "finding decoy code", "gathering context for the quiz", question categories, question count, or anything that reveals the quiz structure. Go directly from the analysis to the `AskUserQuestion` call.

Generate all questions up front, then present them **all at once in a single `AskUserQuestion` call** using the multi-question feature. This renders as a panel where the user navigates between questions with arrow keys and submits all answers together.

Each question in the `questions` array should have:
- `question`: The full question text, prefixed with difficulty (e.g. "[Basic] What is..."). For decoy questions, include the code snippet directly in the question text.
- `header`: Short label like "Q1", "Q2", "Q3", "Q4"
- `options`: 2-4 choices with `label` (the answer text) and `description` (empty string is fine). One correct, the rest plausible but wrong. Randomize the position of the correct answer.
- `multiSelect`: false

Do NOT output any text before the `AskUserQuestion` call. The questions themselves are the first thing the user should see after the diff analysis.

#### Question Categories

Pick from these based on relevance to the diff. Each question should reference specific file names, function names, or module names from the changes — not generic concepts.

- **Intent & Architecture** (at least 1): Why this approach over alternatives. What pattern was introduced and why.
- **Blast Radius** (at least 1): What breaks if something goes wrong — which modules, endpoints, flows are affected.
- **Deployment & Infrastructure** (if relevant): Migration concerns, config changes, deployment order.
- **Risk & Edge Cases** (at least 1): Race conditions, backwards compatibility, failure modes, rollback plan.
- **Verification** (if room): What test or check would catch a regression.

#### Decoy Question (exactly 1 per quiz)

Include one trick question that presents a real code snippet from the repo that was **NOT changed** in the PR, framed as if it were part of the change. The correct answer is always a variant of "This code was not modified in this PR."

Pick code that is thematically related but untouched — same directory or similar domain. Include the code snippet directly in the `question` text. Include 2-3 plausible reasons the code *could* have changed alongside the correct "not modified" option. The decoy catches developers who are guessing without having read the diff.

The decoy counts toward the total question count. Place it at a random position — not always last.

To prevent the "not modified" option from being an obvious tell, include it as a wrong answer in 1-2 of the non-decoy questions too. For those questions, the code *was* changed, so "not modified" is incorrect.

#### Difficulty Mix

- 1 Basic (skimming the diff is enough)
- 1-2 Intermediate (requires understanding the architecture)
- 1-2 Advanced (requires understanding tradeoffs and failure modes)

Prefix each question with number and difficulty: "Q1/4 [Basic] — ..."

After receiving all answers, evaluate them together and present results in Step 5.

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

Claude calls `AskUserQuestion` with 3 questions in a single call:

```json
{
  "questions": [
    {
      "question": "[Basic] This PR switches session handling from JWT in localStorage to httpOnly cookies. What is the primary security benefit?",
      "header": "Q1",
      "options": [
        { "label": "Prevents XSS attacks from accessing tokens", "description": "" },
        { "label": "Reduces server memory usage", "description": "" },
        { "label": "Enables faster token validation", "description": "" }
      ],
      "multiSelect": false
    },
    {
      "question": "[Intermediate] The function refresh_token() in auth/tokens.py was also updated in this PR. What was the purpose of the change?",
      "header": "Q2",
      "options": [
        { "label": "Added retry logic for expired refresh tokens", "description": "" },
        { "label": "This code was not modified in this PR", "description": "" },
        { "label": "Switched from symmetric to asymmetric signing", "description": "" }
      ],
      "multiSelect": false
    },
    {
      "question": "[Advanced] If the cookie-based session is deployed before the frontend removes the localStorage token read, what happens?",
      "header": "Q3",
      "options": [
        { "label": "Users get logged out on every page load", "description": "" },
        { "label": "Both tokens are sent, but the cookie takes precedence", "description": "" },
        { "label": "The old localStorage token is used, bypassing httpOnly protection", "description": "" },
        { "label": "This code was not modified in this PR", "description": "" }
      ],
      "multiSelect": false
    }
  ]
}
```

The user navigates between Q1/Q2/Q3 with arrow keys, selects answers, and submits all at once. Claude then presents results in Step 5.
