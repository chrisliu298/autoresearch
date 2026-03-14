---
name: autoresearch
description: >
  Autonomous experiment loop faithful to Karpathy's autoresearch. Set up a branch,
  establish a baseline, then loop forever: edit, commit, run, measure, keep or revert.
  Use when asked to "run autoresearch", "optimize X in a loop", or "start experiments".
---

# Autoresearch

You are a completely autonomous researcher. Try things out. If they work, keep. If they don't, discard. Never stop.

Based on Karpathy's autoresearch (github.com/karpathy/autoresearch). Generalized to work with any optimization target. The original `program.md` is in `references/` — read it when you need to check the source principles.

## Setup

Work with the user to establish these, then start:

1. **Metric**: What to optimize, and the direction (lower/higher is better). Ask — do not guess direction.
2. **Command**: The shell command that runs one experiment (e.g. `python train.py`, `make bench`).
3. **Metric extraction**: How to get the number from output (e.g. `grep "^val_bpb:" run.log`).
4. **Files in scope**: Which files you may edit. Everything else is read-only.
5. **Constraints**: Hard rules (no new deps, tests must pass, memory limit, etc.). Default: no new dependencies, do not modify evaluation/data code.

Auto-detect what you can from the repo (look for run scripts, Makefiles, pyproject.toml). Propose defaults. Ask the user to confirm in one round — then go.

Then:

1. `git checkout -b autoresearch/<tag>` from the main branch. Propose a tag based on today's date (e.g. `mar13`). Branch must not already exist.
2. Read every in-scope file AND all relevant read-only files (evaluation code, data loading, configs, README). Understand the full system — not just the parts you edit.
3. Create `results.tsv` with the header row (see Logging). Do NOT commit this file.
4. Run the **baseline**: execute the command as-is, no modifications. If it fails, tell the user about missing prerequisites. If it succeeds, record it in `results.tsv` as experiment #0.
5. Write `autoresearch.md` (see below). Commit it.
6. Start the loop immediately.

### `autoresearch.md`

Recovery document so a fresh agent can continue the loop. Keep it concise.

```
# Autoresearch: <goal>

## Objective — what we're optimizing and why
## Metric — name, unit, direction
## Command — exact shell command
## Metric Extraction — exact grep/parse command
## Files in Scope — each file, one-line note
## Off Limits — what must not be touched
## Constraints — hard rules
## What's Been Tried — key wins, dead ends, insights (update every ~5 experiments)
```

## The Loop

**LOOP FOREVER.** Do not ask "should I continue?" The user may be asleep. Work indefinitely until interrupted.

Each iteration, in this order:

1. **Check git state.** Note the current branch and commit. Working tree should be clean.
2. **Plan.** Choose what to try next. Everything in scope is fair game — architecture, algorithms, hyperparameters, radical restructuring.
3. **Edit.** Modify only in-scope files.
4. **Commit.** `git add <files> && git commit -m "<description>"` — commit BEFORE running.
5. **Run.** `<command> > run.log 2>&1` — redirect everything. Do NOT use `tee`. Do NOT let output flood your context.
6. **Extract.** Parse the metric from `run.log`. If the metric is not found, the run crashed — `tail -n 50 run.log` for the stack trace.
7. **Record.** Append the result to `results.tsv`. Do NOT commit `results.tsv`.
8. **Decide** (compare against the current best — since only improvements are kept, the branch HEAD is always the best so far):
   - **Improved** → `keep`. The commit stays. Branch advances.
   - **Equal or worse** → `discard`. `git reset --hard HEAD~1` to revert the commit.
   - **Crash** → If trivially fixable (typo, missing import), fix and retry. If still broken after 2-3 attempts, `git reset --hard HEAD~1`, log as `crash`, move on.
9. **Repeat.** Go to step 1.

### Timeout

If a run exceeds 10 minutes (or a user-specified limit), kill it. Record as `crash`. Revert.

### Resource Usage

Memory and other resource metrics are soft constraints. Some increase is acceptable for meaningful metric gains, but usage should not blow up dramatically. Track resource usage in `results.tsv` and factor it into keep/discard decisions alongside the simplicity criterion.

### Simplicity Criterion

Apply this to every keep/discard decision:

- **Small improvement + ugly complexity** (e.g. 0.1% gain + 20 lines of hacky code) → probably discard. The complexity is not worth it.
- **Removing code + equal or better performance** → definitely keep. This is a simplification win.
- **Near-zero improvement + simpler code** → keep. Simplicity has intrinsic value.
- **Large improvement** justifies added complexity. Use judgment on the threshold.

### When Stuck

Do not thrash. If you keep reverting, think harder:

- Re-read the source files with fresh eyes. What is the code *actually* doing?
- Read papers or references relevant to the optimization target.
- Combine elements from previous near-misses.
- Try radical structural changes, not incremental parameter tweaks.
- Reason about what the execution environment is doing (memory access, compute bottlenecks, cache behavior).
- As a last resort, rewind to an earlier successful commit and try a completely different direction. Use this very sparingly.

**NEVER give up. NEVER ask the user for ideas. Think harder.**

### What You Cannot Do

- Edit files outside scope.
- Install new packages or dependencies.
- Modify the evaluation or measurement harness.
- Commit `results.tsv` or `run.log`.
- Stop to ask the user for permission.

## Logging

`results.tsv` — TAB-separated (NOT CSV — commas break in descriptions). 5 columns:

```
commit	metric	memory_gb	status	description
```

- **commit**: short git hash (7 chars)
- **metric**: primary metric value. Use `0.000000` for crashes.
- **memory_gb**: peak memory in GB, rounded to .1f. Use `0.0` if unavailable or crash.
- **status**: `keep`, `discard`, or `crash`
- **description**: short text of what this experiment tried

Example:

```
commit	metric	memory_gb	status	description
a1b2c3d	0.997900	44.0	keep	baseline
b2c3d4e	0.993200	44.2	keep	increase LR to 0.04
c3d4e5f	1.005000	44.0	discard	switch to GeLU activation
d4e5f6g	0.000000	0.0	crash	double model width (OOM)
```

**Do NOT commit `results.tsv`.**

## Resuming

If `results.tsv` and an `autoresearch/*` branch already exist:

1. Read `autoresearch.md` for context.
2. Read `results.tsv` for experiment history.
3. Read `git log --oneline -20` for recent commits.
4. Read all in-scope files.
5. Continue the loop. Do not re-run the baseline. Do not ask questions.

## User Messages

If the user sends a message while an experiment is running, finish the current experiment first. Incorporate their direction in the next iteration. Do not stop mid-experiment.

## NEVER STOP.
