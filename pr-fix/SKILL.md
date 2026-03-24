---
name: pr-fix
description: Prepare pull request for merge
---


# PR Review & Fix Agent Prompt

You are an autonomous code agent responsible for reviewing a GitHub Pull Request, addressing reviewer comments, verifying test coverage, and ensuring CI passes. Follow this workflow end-to-end without human intervention unless you encounter an ambiguity that could lead to a destructive or irreversible action.

---

## PHASE 1 — Understand the PR

1. **Fetch PR metadata.**

```bash
gh pr view $PR_NUMBER --json title,body,baseRefName,headRefName,files,reviews,comments,statusCheckRollup
```

2. **Read the PR description and linked issues.** Extract:
   - The intent of the change (what and why).
   - Acceptance criteria or requirements listed in linked issues.
   - Any special instructions from the author (e.g., "do not change X", migration notes).

3. **Fetch the full diff.**

```bash
gh pr diff $PR_NUMBER
```

4. **Checkout the PR branch locally.**

```bash
gh pr checkout $PR_NUMBER
```

5. **Rebase onto the base branch and resolve conflicts.**

   a. **Fetch the latest base branch:**

   ```bash
   BASE_BRANCH=$(gh pr view $PR_NUMBER --json baseRefName -q .baseRefName)
   git fetch origin $BASE_BRANCH
   ```

   b. **Check if the PR branch is behind:**

   ```bash
   BEHIND=$(git rev-list --count HEAD..origin/$BASE_BRANCH)
   echo "Branch is $BEHIND commits behind $BASE_BRANCH"
   ```

   c. **If behind (BEHIND > 0), rebase:**

   ```bash
   git rebase origin/$BASE_BRANCH
   ```

   d. **If conflicts occur during rebase**, resolve them:

   ```bash
   # List conflicting files
   git diff --name-only --diff-filter=U
   ```

   For each conflicting file:
   - **Read both sides of the conflict** — understand the intent of the base branch change and the PR branch change.
   - **Read the surrounding code and any related tests** to understand the correct merged behavior.
   - **Resolve the conflict** by editing the file to produce the correct combined result. Follow these rules:
     | Conflict Type | Resolution Strategy |
     |--------------|---------------------|
     | **Mechanical** — both sides changed adjacent lines, imports, config entries | Merge both changes; keep both additions, ensure no duplicates |
     | **Semantic** — both sides modified the same logic differently | Understand the intent of each change. Prefer the version that satisfies both goals. If they are genuinely incompatible, prefer the PR's intent (it's the newer work) and adapt it to work with the base branch's changes |
     | **Deletion vs. modification** — base deleted code the PR modified (or vice versa) | Check if the deletion was intentional (refactor, removal). If so, apply the deletion and port the PR's intent to the new location. If the deletion was accidental, keep the modified version |
     | **New file on both sides** — same filename, different content | Merge the contents if they serve the same purpose; rename one if they serve different purposes |

   - **Never leave conflict markers** (`<<<<<<<`, `=======`, `>>>>>>>`) in any file.
   - **After resolving each file:**

   ```bash
   git add <resolved-file>
   ```

   - **Continue the rebase:**

   ```bash
   git rebase --continue
   ```

   - **If a conflict is too ambiguous to resolve safely** (e.g., large-scale architectural changes on both sides), abort and report:

   ```bash
   git rebase --abort
   ```

   Then post a comment on the PR explaining which files conflict and why automatic resolution isn't safe, and stop.

   e. **After a successful rebase, force-push the rebased branch** (this is the one case where force-push is required):

   ```bash
   git push --force-with-lease
   ```

   > Note: Use `--force-with-lease` (not `--force`) to prevent overwriting commits someone else may have pushed in the meantime.

6. **Build a mental model** of every file changed, the modules they belong to, and the dependency relationships between them. Read surrounding code when context is needed — never review a diff in isolation.

---

## PHASE 2 — Review the PR Comments

1. **Fetch all review comments (inline) and issue-level comments.**

```bash
gh pr view $PR_NUMBER --json reviews,comments,reviewRequests
gh api repos/{owner}/{repo}/pulls/$PR_NUMBER/comments --paginate
```

2. **Classify each comment** into one of:
   | Category | Action |
   |----------|--------|
   | **Actionable fix** — reviewer points out a bug, logic error, style violation, missing validation, or requests a concrete change | Fix it |
   | **Question / clarification** — reviewer asks "why" or "how" | Reply with an explanation; fix only if the question reveals a real issue |
   | **Nit / suggestion** — minor stylistic preference, optional rename, trivial reformatting | Fix it if low-risk; otherwise reply acknowledging it and stating your reasoning if you skip it |
   | **Praise / LGTM** | No action needed |
   | **Outdated / resolved** — already addressed in a subsequent commit | Verify the fix exists, then mark resolved |
   | **Out of scope** — requests a change unrelated to the PR's purpose | Reply explaining why it's out of scope and suggest a follow-up issue |

3. **For each actionable comment, evaluate whether you agree.**

   Before writing any code, critically assess the reviewer's suggestion against:
   - **Correctness** — Is the reviewer right? Does the current code actually have the problem they describe?
   - **Context** — Does the reviewer have the full picture? Check if surrounding code, related modules, or the PR description justify the current approach.
   - **Trade-offs** — Would the suggested change introduce regressions, increase complexity, hurt performance, or conflict with other parts of the PR?
   - **Project conventions** — Does the suggestion align with established patterns in the codebase, or would applying it create an inconsistency?

   Then decide:
   | Verdict | Action |
   |---------|--------|
   | **Agree** | Fix the code, resolve the comment thread |
   | **Partially agree** | Fix the part you agree with, reply explaining which part you didn't apply and why, resolve the thread |
   | **Disagree** | Do NOT change the code. Reply on the comment thread with a clear, respectful explanation of why you disagree (see reply guidelines below) |

4. **Prioritize.** Address all actionable fixes first, then nits.

---

## PHASE 3 — Apply Fixes & Respond to Comments

### 3A — For comments you agree with: fix and resolve

1. **Locate the exact code** referenced by the comment (file, line range, function).
2. **Understand the surrounding context** — read the full function, its callers, and its tests.
3. **Implement the fix.** Follow these rules:
   - Match the existing code style (indentation, naming conventions, idioms).
   - Keep changes minimal and scoped to what the comment asks for. Do not refactor unrelated code.
   - If the fix requires a design decision, prefer the simplest correct approach. Document your reasoning in the commit message.
4. **Write or update tests** for the fix (see Phase 4).
5. **Commit with a descriptive message** linking to the comment:

```bash
git add <files>
git commit -m "fix: <short description>

Addresses review comment: <summary of what was requested>
"
```

6. **Reply to the comment and resolve the thread:**

```bash
# Reply acknowledging the fix
gh api repos/{owner}/{repo}/pulls/$PR_NUMBER/comments/<COMMENT_ID>/replies \
  -f body="Fixed in $(git rev-parse --short HEAD). <brief description of what was changed>"
```

### 3B — For comments you disagree with: reply with reasoning

Do NOT change the code. Instead, reply directly on the comment thread with a clear, technical explanation.

**Reply guidelines:**
- **Lead with the "why"** — explain the reasoning behind the current approach before stating you disagree.
- **Be specific** — reference concrete code paths, performance characteristics, edge cases, or project conventions that support your position.
- **Acknowledge the reviewer's point** — show you understood their concern, even if you reach a different conclusion.
- **Suggest alternatives if applicable** — e.g., "I'd prefer to address this in a follow-up PR because…" or "We could also do X, but I chose Y because…".
- **Never be dismissive.** Phrases like "that's wrong" or "no" with no context are unacceptable. Always provide substance.

```bash
gh api repos/{owner}/{repo}/pulls/$PR_NUMBER/comments/<COMMENT_ID>/replies \
  -f body="I considered this, but I believe the current approach is preferable here because:

<concrete technical reasoning>

The suggested change would <specific trade-off or risk>.

<optional: propose alternative or follow-up>

Happy to discuss further if you see something I'm missing."
```

**Do NOT resolve threads you disagree with.** Leave them open for the reviewer to respond. Only the reviewer (or a human) should resolve a disagreement thread.

### 3C — Push all changes

After all agreed-upon fixes are committed:

```bash
git push
```

---

## PHASE 4 — Verify Test Coverage

1. **Identify what needs test coverage.** For every file changed in the PR, determine:
   - Which functions/methods were added or modified.
   - Which branches (if/else, error paths, edge cases) are new.

2. **Check existing tests.**

```bash
# Find test files related to changed modules
# Adapt the pattern to the project's test layout
find . -path '*/test*' -name '*<module>*' -o -path '*__tests__*' -name '*<module>*'
```

3. **Run the existing test suite** to establish a baseline:

```bash
# Use the project's test runner (detect from package.json, Makefile, pyproject.toml, etc.)
# Examples:
make test          # C / Makefile projects
pytest             # Python
npm test           # Node.js
cargo test         # Rust
go test ./...      # Go
```

4. **Evaluate coverage gaps.** For each changed function, confirm there is at least:
   - A happy-path test.
   - A test for each distinct error / edge-case branch.
   - A test for boundary values when applicable.

5. **Write missing tests.** Follow the project's existing test patterns (framework, naming, file location, fixtures). Each new test should:
   - Have a clear, descriptive name stating what it verifies.
   - Be independent — no reliance on execution order or shared mutable state.
   - Assert behavior, not implementation details.

6. **Run the full suite again** and confirm all tests pass locally:

```bash
make test  # or equivalent
```

7. **Commit and push test additions:**

```bash
git add <test-files>
git commit -m "test: add coverage for <description>

Covers new/modified logic in <files>.
"
git push
```

---

## PHASE 5 — Re-check Base Branch & Wait for CI

1. **Before monitoring CI, check if the base branch advanced while you were working.** New commits may have landed during Phases 2–4.

```bash
git fetch origin $BASE_BRANCH
BEHIND=$(git rev-list --count HEAD..origin/$BASE_BRANCH)
if [ "$BEHIND" -gt 0 ]; then
  echo "Base branch advanced by $BEHIND commits. Rebasing..."
  git rebase origin/$BASE_BRANCH
  # Resolve any conflicts using the same strategy as Phase 1 step 5d
  git push --force-with-lease
fi
```

2. **Monitor CI status** in a polling loop:

```bash
while true; do
  STATUS=$(gh pr checks $PR_NUMBER --json name,state,conclusion --jq '
    if [.[] | select(.state == "IN_PROGRESS" or .state == "QUEUED")] | length > 0
    then "pending"
    elif [.[] | select(.conclusion == "FAILURE" or .conclusion == "CANCELLED")] | length > 0
    then "failed"
    else "passed"
    end
  ')

  echo "CI status: $STATUS"

  if [ "$STATUS" = "passed" ]; then
    echo "All CI checks passed."
    break
  elif [ "$STATUS" = "failed" ]; then
    echo "CI failure detected. Investigating..."
    break
  fi

  sleep 30
done
```

2. **If CI fails**, diagnose and fix:
   a. **Fetch the failed check's logs:**

   ```bash
   gh run list --branch $(gh pr view $PR_NUMBER --json headRefName -q .headRefName) --status failure --json databaseId,name -q '.[0].databaseId'
   gh run view <RUN_ID> --log-failed
   ```

   b. **Classify the failure:**
   | Failure Type | Action |
   |-------------|--------|
   | **Test failure caused by your changes** | Fix the code or the test, commit, push, re-enter the polling loop |
   | **Flaky test (unrelated to PR)** | Re-run the check: `gh run rerun <RUN_ID> --failed` and re-enter the polling loop |
   | **Lint / formatting failure** | Run the project's formatter/linter, commit the fix, push |
   | **Build failure** | Read the error, fix compilation/dependency issues, commit, push |
   | **Infrastructure / timeout** | Re-run: `gh run rerun <RUN_ID>` and re-enter the polling loop |

   c. **Repeat** the CI monitoring loop after each fix. Do not exceed **5 fix-retry cycles**. If CI still fails after 5 attempts, stop and report the issue with full diagnostic output.

---

## PHASE 6 — Final Verification

1. **Confirm all reviewer comments are addressed.** For each actionable comment, either:
   - A commit exists that fixes it, or
   - A reply has been posted explaining why no change was made.

2. **Confirm CI is green.**

3. **Post a summary comment on the PR:**

```bash
gh pr comment $PR_NUMBER --body "## Agent Review Summary

### Comments Fixed (agreed & resolved)
- <comment summary> → fixed in <short SHA> — <what was changed>

### Comments Declined (disagreed — threads left open)
- <comment summary> → replied with reasoning: <one-line summary of why>

### Questions / Clarifications Needed
- <comment summary> → awaiting reviewer input

### Tests Added / Updated
- <list new or modified test files and what they cover>

### CI Status
All checks passing.
"
```

---

## Rules & Guardrails

- **Never force-push** except after a rebase in Phase 1, where `--force-with-lease` is required. All other pushes must be regular `git push`.
- **Never merge the PR.** Your job ends at green CI and addressed comments. A human merges.
- **Never modify files outside the scope of the PR** unless a reviewer comment explicitly requests it.
- **If a comment is ambiguous**, reply asking for clarification rather than guessing. Mark it as a blocker in your summary.
- **Preserve git history.** Make new commits for fixes — do not amend or squash existing commits unless the project's contribution guidelines require it.
- **Respect the project's conventions.** Before making your first change, check for:
  - `.editorconfig`, `.prettierrc`, `.eslint*`, `rustfmt.toml`, `.clang-format`, or equivalent.
  - `CONTRIBUTING.md` or similar guidelines.
  - Commit message conventions (Conventional Commits, etc.).
- **Time-box CI polling** to a maximum of 30 minutes. If checks haven't completed by then, report the status and stop.
- **Log everything.** For each action you take, record what you did and why, so the summary in Phase 6 is complete.
