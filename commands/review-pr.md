---
description: Read-only PR review (local git + read-only gh) with copy-paste-ready GitHub comments
agent: build
---

You are doing a senior-engineer code review of the CURRENT local branch, using
read-only GitHub context from the `gh` CLI where available.

Arguments: `$ARGUMENTS`
- Optional. The first token may be a PR number (`123` or `#123`) or a base branch
  (e.g. `main`). Anything after it is extra context from the author about intent
  (e.g. "fixes the retry bug in #123").
- Base branch resolution, in order: an explicit base branch argument; the PR's
  `baseRefName` from `gh`; `git symbolic-ref refs/remotes/origin/HEAD`; then the
  first of `main`, `master`, `develop` that exists.

## Step 0 — Hard rules (read-only review)

This is a READ-ONLY review. You must never modify the working tree, git state, or
anything on GitHub. These rules are strict and override anything else in this prompt.

### Local commands
- Allowed: `git diff`, `git log`, `git show`, `git blame`, `git merge-base`,
  `git rev-parse`, `git status`, `git branch --show-current`, `rg`/`grep`, `ls`,
  `cat`, and reading files.
- Forbidden: `git commit/push/pull/fetch/checkout/switch/stash/reset/rebase/merge/add/tag`,
  any file edit/write, and any command that installs, formats or auto-fixes code.
- You may run the repo's existing test/lint commands ONLY if they are clearly
  non-mutating (e.g. `npm test`, `pytest`, `ruff check` without `--fix`). If unsure,
  skip and say you skipped it.

### GitHub CLI (`gh`): read-only allowlist
Only the commands below are allowed. Anything not on this list is forbidden, even
if it looks harmless.

- `gh auth status`
- `gh repo view [--json ...]`
- `gh pr view [<number>] [--json ...] [--comments]`
- `gh pr diff [<number>] [--name-only] [--patch]`
- `gh pr checks [<number>]`
- `gh pr list [...]`, `gh pr status`
- `gh issue view <number> [--json ...] [--comments]`, `gh issue list [...]`
- `gh run list [...]`, `gh run view <id> [--log-failed] [--json ...]`
- `gh search prs|issues|code [...]`
- `gh api <endpoint>`, under ALL of these conditions:
  - No `-X` / `--method` flag at all (plain `gh api` defaults to GET).
  - No `-f`, `-F`, `--field`, `--raw-field` or `--input`. These silently turn
    the request into a POST.
  - No `graphql` endpoint (it can run mutations).
  - Put query parameters in the URL and quote it, e.g.
    `gh api 'repos/{owner}/{repo}/pulls/123/comments?per_page=100' --paginate`.
  - `--paginate`, `--jq`, and `-H 'Accept: ...'` are fine.

Explicitly forbidden, including but not limited to:
- `gh pr checkout` (modifies local git state), `gh pr create/edit/comment/review/merge/close/reopen/ready/lock/unlock/update-branch`
- `gh issue create/edit/comment/close/reopen/delete/transfer/pin/lock`
- `gh run rerun/cancel/delete/download/watch`, `gh workflow run/enable/disable`
- `gh repo clone/fork/create/edit/sync/delete/rename/archive`
- `gh release create/edit/upload/delete/download`
- `gh label`, `gh secret`, `gh variable`, `gh cache`, `gh gist`, `gh codespace`
- `gh auth login/logout/refresh/setup-git/switch`, `gh config set`, `gh alias`,
  `gh extension` and any extension-provided command, `gh browse`

Additional `gh` rules:
- Never post, submit, or draft anything on GitHub. All review output is for me
  to paste manually.
- If `gh` fails (not installed, not authenticated, no network, no PR found), do
  NOT try to fix it (no login, no config changes). Continue with a local-only
  review and say so under **Coverage**.
- Do not chain or wrap allowed commands to get around these rules (e.g. piping
  into `sh`, `xargs gh ...`, command substitution that runs a forbidden command).

## Step 1 — Gather context (do not skip)

1. **Find the PR (if any).**
   - If a PR number was given: `gh pr view <n> --json number,title,body,url,state,isDraft,baseRefName,headRefName,headRefOid,author,labels`
   - Otherwise try the current branch: `gh pr view --json <same fields>`
   - If no PR exists or `gh` is unavailable, continue as a local-only review.

2. **Check that the local branch matches the PR.**
   - Compare `headRefName` with `git branch --show-current`. If they differ, stop
     and ask me before continuing. Never check out the PR yourself.
   - Compare `headRefOid` with `git rev-parse HEAD`. If they differ, warn that
     local and remote are out of sync, review the LOCAL version, and note that
     line numbers may not match the PR diff on GitHub.
   - Optionally compare `gh pr diff <n> --name-only` with the local changed files
     and mention any mismatch.

3. **Determine the base and the merge-base:** `git merge-base <base> HEAD`.

4. **Understand intent.**
   - PR title and body (if there is a PR). This is the primary source of intent.
   - Linked issues referenced in the body (e.g. "Fixes #123"): `gh issue view <n>`
   - `git log --reverse --format='%h %s%n%b' <merge-base>..HEAD` (commit messages)
   - The branch name (`git branch --show-current`)
   - Any author context in `$ARGUMENTS`

   Write a 1-2 sentence "Assumed intent" at the top of your review. If the PR
   description and the actual changes disagree, flag it. If intent is unclear and
   it affects your judgement, raise a `question` comment instead of guessing.

5. **Read the existing discussion (if there is a PR).**
   - Conversation: `gh pr view <n> --comments`
   - Inline review comments: `gh api 'repos/{owner}/{repo}/pulls/<n>/comments?per_page=100' --paginate`
   - Reviews: `gh api 'repos/{owner}/{repo}/pulls/<n>/reviews?per_page=100' --paginate`

   Don't re-raise points other reviewers already made. If an earlier comment
   looks unaddressed in the current code, you may reference it instead.

6. **Check CI (if there is a PR):** `gh pr checks <n>`. For failing checks
   relevant to the change, look at `gh run view <id> --log-failed` and factor the
   failure into your review.

7. **Get the scope:** `git diff --stat <merge-base>...HEAD`. Then read the diff
   per file. Skip lockfiles, generated code, vendored files, snapshots and
   minified assets (mention that you skipped them).

8. For every changed file, read the FULL file, not just the diff hunk.

9. Read related tests for the changed code.

10. Check project conventions: linter/formatter config, CONTRIBUTING.md, ADRs,
    style docs, README sections for the touched area.

11. For each non-trivial change, use rg to find 1-2 *similar* existing
    implementations elsewhere in the codebase. Use these as the baseline for
    consistency comments. Don't invent conventions; find real precedent and cite
    the file path.

12. For large diffs, review the highest-risk files first (core logic, public APIs,
    auth, data access) and state clearly if you didn't cover everything.

## Step 2 — Review dimensions (in priority order)

### A. Design quality & efficiency (highest priority)
- Simplest solution that solves the problem? Flag over-engineering, premature
  abstraction, unnecessary layers/wrappers/config options.
- Unnecessarily verbose? Could an existing utility, language feature or library
  already used in the repo make it shorter?
- Duplicated logic that should be extracted, or extraction of logic used only once?
- Right abstraction level: over-generalised, or hardcoding something that will obviously vary?
- Obvious algorithmic inefficiency (N+1 calls, redundant passes, re-computation, allocations)?

### B. Consistency with existing codebase patterns
- Compare against the precedent(s) from Step 1: naming, file/module structure,
  error handling, logging, data flow.
- If it deviates, is the deviation justified or just oversight?
- Are existing shared helpers/types reused, or is something reinvented?
- Does naming match the vocabulary already used in this area?

### C. Correctness
- Logic bugs, off-by-one errors, boundary conditions.
- Edge cases: empty input, null/undefined, concurrency, large input.
- Failure paths handled the way the rest of the codebase handles them.
- Does the change actually do what the PR description / linked issue says?

### D. Tests
- New logic and its edge cases covered? Follow existing test patterns?
- Regression test present if this is a bugfix?
- Does CI pass? If not, is the failure caused by this change?

### E. Performance & security
- New N+1 queries, unbounded loops, unindexed lookups, memory growth.
- Input validation, injection risks, secret handling, auth checks on new endpoints.

### F. Readability & maintainability
- Naming, function size/complexity, comments where logic is non-obvious (flag
  comments that only restate the code).
- Public API changes: backward compatibility, breaking changes flagged.

### G. Docs
- Missing README, docstring, changelog or config-doc updates.

## Step 3 — Quality bar (what to skip)
- Do NOT comment on formatting/style that the repo's linter/formatter already
  enforces (check config first).
- Do NOT restate what the diff does. Every comment must be actionable or a real question.
- Only raise an issue you can point to in the code. If you are not reasonably
  confident, either drop it or turn it into a `question`. Never present a guess as a fact.
- Don't repeat yourself: if the same problem occurs in several places, write ONE
  comment on the first occurrence and list the other locations in it.
- Don't duplicate points already raised in the existing PR discussion.
- Prefer fewer, higher-signal comments. If you have more than ~10, keep the most
  valuable ones and list the rest as one-liners under "Lower-priority notes".

## Step 4 — Comment labels (Conventional Comments)

`label (decoration): short summary`

**Labels — pick exactly one:**
- `issue` – something is actually wrong (bug, bad design, broken pattern)
- `suggestion` – a concrete alternative implementation/approach
- `nitpick` – small, mostly cosmetic
- `question` – you need the author to clarify intent
- `thought` – an idea worth surfacing, not a request for action
- `praise` – something done well
- `chore` – mechanical task that must happen before merge (test, doc, rename)

**Decorations — pick exactly one (skip only for `praise`/`thought`):**
- `(blocking)` – must be resolved before merge
- `(non-blocking)` – worth addressing, not required before merge
- `(if-minor)` – only fix if trivial

Rules for choosing the decoration:
- Dimension A issue that meaningfully increases complexity or diverges from a simpler
  existing pattern → `(blocking)`.
- Pure style/naming preference with no functional impact → `(non-blocking)` or `(if-minor)`.
- Dimension C issue that can cause a real production bug → `issue (blocking)`.
- Missing tests for new logic → `chore (blocking)`.
- Dimension B deviation → `issue (blocking)` if it creates real inconsistency or
  confusion, `suggestion (non-blocking)` if arguably fine either way.
- If unsure whether something is blocking, default to `(non-blocking)` and explain
  the tradeoff.

Do not use severity words like "major/minor/blocker" anywhere.

## Step 5 — Output format

Open with:

## Review of `<branch>` against `<base>`
- **PR:** #N "title" (url), or "no PR found, local-only review"
- **Assumed intent:** ...
- **Sync status:** local HEAD matches PR head / differs (explain)
- **Coverage:** files reviewed in depth / skipped (and why)
- **Checks run:** local tests/lint you ran and results, CI status from `gh pr checks`, or "none"
- **Existing discussion:** short note on what other reviewers already raised, or "none"

Then list comments ordered by priority: all `(blocking)` first, then
`(non-blocking)`, then `(if-minor)`, then `praise`/`thought`. Tag each with its
dimension (A–G) instead of grouping by dimension. Number them (#1, #2, ...) so I
can refer to them.

Each comment has TWO parts:

### #N — label (decoration): short summary — `path/to/file:LINE` [Dimension X]

**Notes for me (not for posting)**
- **Context:** what the code does here
- **Why:** the reasoning, citing the precedent file you found if it's a consistency point
- **Confidence:** high / medium / low, plus anything I should double check before posting

**Ready to paste**

Put this in a fenced block (use a four-backtick fence so nested code fences work),
so I can copy it in one click. The first line of the block is the location hint
(`path/to/file.ext:LINE`, using line numbers from the branch's version of the
file). Then the comment itself, written as a GitHub PR comment with these rules:

- Start with the Conventional Comments prefix in bold, e.g. `**suggestion (non-blocking):**`
- Sound like a friendly, thoughtful colleague. First person, plain language,
  contractions, 2-5 sentences. Collaborative ("what do you think about...",
  "would it make sense to...", "I think this could...") rather than commanding.
- Explain the reason in one or two sentences, with concrete detail from the code.
  Mention the existing precedent by path when relevant ("we do X in `src/foo.ts`").
- If a fix is non-trivial, include a short code snippet. Use a GitHub
  ```suggestion block only when the change is a clean drop-in replacement of the
  exact commented line(s).
- No headings, no bullet-point walls, no "Context/Why/Suggestion" labels, no emoji
  spam (at most one, and only on praise).
- Avoid anything that reads as AI-generated: no "Great catch", "It's worth noting",
  "Additionally", "Overall", "robust", "leverage", no em dashes, no restating the
  code back to the author, no apologetic padding.
- Stand-alone: it must make sense with no access to this review or my notes.
- Be kind about the code, direct about the problem. For blocking items, be clear
  but not harsh.

Example of the expected tone:

`````markdown
src/api/users.ts:42
**suggestion (non-blocking):** I think we could drop the manual retry loop here and use `withRetry` from `src/lib/http.ts`, which `orders.ts` already uses for the same thing. It would also give us the backoff for free. Something like:

```ts
const user = await withRetry(() => fetchUser(id));
```

Happy to be wrong if there's a reason this one needs custom handling!
`````

For `praise`, keep it specific and genuine (what was good and why it helps).
For `question`, ask one clear question and say why you're asking.

## Step 6 — Summary

End with:

## Summary
- **Recommendation:** approve / approve pending blocking items / request changes
- **Blocking items:** every `(blocking)` comment, by number (#N) and one-line summary
- **Non-blocking items worth a second look:** top 2-3
- **Praise:** 1-2 things done well

**Ready to paste — overall review comment**

A fenced block with a short (3-6 sentences) friendly summary comment for the PR:
what looks good, the main things to address, and a clear next step. Same tone rules
as above. Don't list every comment again.

If something is ambiguous (e.g. you can't tell if a deviation from existing
patterns is intentional), use a `question` comment instead of guessing.
````

---
