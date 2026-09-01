# PR Review: The Core Method

One sentence version: **don't trust the diff, don't trust the description, don't trust green CI — verify each claim yourself, cheaply.**

Everything below is that idea applied to different parts of a PR.

---

## Step 0 — Find the *real* diff

Before reading anything, make sure you're looking at the actual change.

```bash
git fetch origin
git log origin/main..HEAD --oneline      # commits actually unique to this branch
git diff origin/main...HEAD --stat        # files actually touched
```

Don't diff against your local `main` — it can be stale and pull in dozens of
unrelated already-merged commits, which drowns the real change in noise.
This bit us today: local `main` was 4 PRs behind `origin/main`.

## Step 1 — Understand intent before code

Read the commit messages / PR description first. Ask: **what bug or need
is this solving, and what's the smallest change that solves it?** Anything
in the diff that doesn't serve that purpose is scope creep — flag it.

## Step 2 — Read the diff like a story, not a checklist

Go file by file, but keep asking two questions per hunk:

- **Why does this line exist?** (if you can't answer, that's a question for the author)
- **What's the failure case this change is guarding against, or introducing?**

## Step 3 — Verify the claim, don't just read it

This is the "ninja technique" — the thing that separates a real review from
a skim. A PR description or commit message is a *claim*. Cheap ways to check it:

| Claim in the PR | How to verify it cheaply |
|---|---|
| "fixes bug X" | Reproduce X on the old code, confirm it's gone on the new code |
| "adds a regression test" | Make the test fail first (revert the fix locally), then pass (restore it) — **red/green** |
| "no behavior change" | Diff the actual output/render/API response before and after |
| "bumps dependency to fix Y" | Read the dependency's source diff between the two versions, don't take the changelog's word for it |
| "safe because of Z" | Find Z in the code yourself; don't assume it exists |

Example from a real review today: a commit claimed MUI 7.1.1 called `onClick`
unconditionally on a Chip, fixed in 7.3.11, with a regression test added.
Verifying it took three cheap steps:

```bash
# 1. Read the actual source diff between versions
grep -n "onClick" node_modules/@mui/material/Chip/Chip.js
#  7.1.1:  onClick(event)        <- unconditional call, the bug
#  7.3.11: onClick?.(event)      <- guarded, the fix

# 2. Prove the test is real (not a false-positive that passes either way)
#    swap in the OLD dependency version, rerun just that test
npx vitest run src/test/components/StatusChip.test.jsx
#  -> FAILS on 7.1.1 (reproduces the real crash)

# 3. Restore the fixed version, rerun
npx vitest run src/test/components/StatusChip.test.jsx
#  -> PASSES on 7.3.11
```

That's the whole technique: **red before green.** A test that only shows
green never proved anything — it might pass on both the broken and fixed
code. You only know it's a real regression guard once you've watched it
fail for the right reason.

## Step 4 — Run it yourself

Never approve on "tests passed in CI" alone if you can run it locally in
under a few minutes. Pull the branch, install, run the full suite (not just
the changed files — catch cross-file regressions):

```bash
git checkout <branch>
npm install          # respect the lockfile, don't skip it
npx vitest run        # full suite, not just the touched files
```

## Step 5 — Trace the blast radius

For every change, ask "what else could this affect?" A couple of angles
that are easy to forget:

- **Deployment path** — does this app run via Docker, a build pipeline,
  a lockfile? If so, "what if a teammate runs `npm install` locally and
  gets a different version" is often a non-issue — check how the app is
  actually built/run (`Dockerfile`, `compose.yml`, CI config) before
  flagging a hypothetical risk.
- **Test leakage** — does a global listener/mock/spy get cleaned up even
  when the test *fails* partway through (`onTestFinished`, `afterEach`),
  or only on the happy path? A cleanup that only runs after success can
  leak state into the next test.
- **Partial migrations** — if a PR renames/changes one usage (e.g. an
  ARIA role, a prop, a function signature), grep the whole repo for the
  old pattern to confirm nothing was missed:
  ```bash
  grep -rn 'getByRole("checkbox"' src/test/
  ```

## Step 6 — Say what you actually verified, not what you assume

When you leave a review comment, distinguish between:
- "I read this and it looks right" (weakest — a skim)
- "I ran the tests and they pass" (better — but green isn't proof, see Step 3)
- "I reproduced the bug on old code and confirmed the fix resolves it" (strongest)

Match the confidence in your comment to the level of verification you
actually did. Don't write "100% confirmed" unless you did the work to
back it up — and if someone asks "are you sure?", that's your cue to go
find the missing piece of evidence, not just restate the claim more firmly.

---

## Quick checklist

- [ ] Diffed against the real base (`origin/main`, not stale local `main`)
- [ ] Understand *why* the change exists, not just *what* it does
- [ ] Verified at least one non-trivial claim directly (source, repro, or red/green test)
- [ ] Ran the full test suite locally, not just the touched files
- [ ] Checked for partial migrations (grep for the old pattern repo-wide)
- [ ] Considered blast radius: deployment path, cleanup-on-failure, scope creep
- [ ] Review comment's confidence matches what was actually verified
