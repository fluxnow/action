# AGENTS.md

## PR Autonomy Rules (applies to every PR you open in this repo)

If you opened a PR in this repo, you are responsible for driving it to green + approved. Do this loop automatically after every push:

1. **Wait ~5 minutes** after push for CI to start
2. **Check status**: `gh pr view <PR> --json statusCheckRollup,reviewDecision,isDraft` — repo is inferred from cwd
3. **Check review comments**: inside a repo, `gh api repos/:owner/:repo/pulls/<PR>/comments` — `:owner/:repo` auto-resolves to the current repo. Equivalent shorter form: `gh pr view <PR> --json reviews` (includes review bodies)
4. **If CI red OR unresolved bot comments**:
   - Lint/format/type errors → auto-fix, commit, push
   - Bot review comments (Gemini, CodeQL) → read each, apply fix, commit, **push**, then reply to bot with the fix-commit URL (push first so the link resolves)
   - Harbor 401 on deploy → find the failed run: `RID=$(gh run list --branch $BRANCH --workflow FluxNow --limit 1 --json databaseId,conclusion -q '.[] | select(.conclusion=="failure") | .databaseId')` then `gh run rerun --failed $RID` ONCE (known flake, tracked INF-314)
   - Merge conflicts with main → rebase on `origin/main`, resolve, re-push (git: `git push --force-with-lease`; jj: `jj git push --force` — jj tracks remote bookmarks and prevents accidental overwrites by default)
   - Go back to step 1
5. **If green + no unresolved bots**: `gh pr ready <PR>` (un-draft), post "ready for review" in Linear thread, done.
6. **Give up after 5 iterations** — post a blocker comment on the Linear issue with the error, don't silently stall.

**Only pause and ping a human when**:
- An AC is ambiguous (request clarification in Linear)
- The fix requires schema / contract changes beyond the stated scope
- An architecture decision is needed
- CI fails with the same error 5 times in a row

**Never**:
- Mark a PR ready before CI is green
- Ignore Gemini or CodeQL warnings without explicitly marking them "out of scope" with justification
- Use `--no-verify`, `git push --force` (without `--force-with-lease`), `jj git push --force` (without human sign-off), or similar destructive flags without human approval

