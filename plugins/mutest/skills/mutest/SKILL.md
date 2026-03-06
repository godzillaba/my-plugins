---
description: Run mutation testing on Solidity files to find where test coverage is lacking, analyze surviving mutants, and optionally write missing tests.
argument-hint: <sol-file> [sol-file ...]
disable-model-invocation: true
---

# Mutation Testing Coverage Gap Finder

If $ARGUMENTS is empty, tell the user they need to specify at least one Solidity file path and stop.

## Step 0: Verify the test suite passes

Run `forge test` and wait for it to complete. If any tests fail, show the user which tests failed and offer to comment them out so mutation testing can proceed:

1. List the failing test names and their files.
2. Use AskUserQuestion to ask: "These tests are failing. Should I comment them out so we can proceed with mutation testing?" with options:
   - **Yes — comment them out**: Comment out the failing test functions (wrap each in `/* ... */` with a `// COMMENTED OUT FOR MUTEST — was failing before mutation testing` marker). Then re-run `forge test` to confirm the suite now passes. If it still fails, repeat.
   - **No — stop here**: Stop and let the user fix the tests manually. Do not proceed to Step 1.

Do not proceed to Step 1 until `forge test` exits with 0.

## Step 1: Ask the user everything up front

Before launching anything, use AskUserQuestion to gather all decisions at once:

1. **Auto-fix**: Should surviving mutants be automatically fixed by writing missing tests? (Yes / No)
2. **Loop until clean**: If auto-fix is enabled, should we re-run mutest after fixes are applied, repeating until all mutants are killed or no more progress is made? (Yes / No)
3. **Test command override**: Would you like to override the test command mutest uses? The default is `forge test --optimize false --threads 1 --root <workerDir>`. If yes, provide the command (it runs via `bash -c` with the worker directory as `cwd`). If no, leave blank to use the default.
4. **Report path prefix**: Where should reports be written? Suggest a default of `mutation-report` in the project root. Reports will be written as `<prefix>-N.md` where N is the iteration number (starting at 1). For a single-pass run, only `<prefix>-1.md` is written.

Remember the answers — do not prompt the user again during the run.

## Step 2: Run an iteration

### 2a: Launch mutest

If the user provided a test command override, append `-t '<cmd>'` to all mutest invocations below.

**First iteration**: run with the user's file arguments:
```
npx @godzillaba/mutest@latest $ARGUMENTS [-t '<cmd>']
```

**Subsequent iterations** (only when loop-until-clean is enabled): run with **NO file arguments** to re-test existing mutants against the updated test suite:
```
npx @godzillaba/mutest@latest [-t '<cmd>']
```

> **CRITICAL**: Do NOT pass a file list on subsequent passes. Passing file arguments causes mutest to **regenerate** mutants from scratch via Gambit, destroying the previous `gambit_out/` state.

Use the Bash tool with `run_in_background: true`. Save the task ID.

### 2b: Wait for mutest to finish

### 2c: Read survivors

After mutest completes, read `gambit_out/survivors.json`. This is a JSON array in the same format as gambit's `gambit_results.json`. Each entry has:
- `"id"`: mutant ID (string)
- `"description"`: mutation type
- `"diff"`: the exact code change
- `"original"`: path to original source file

If there are no survivors, skip to Step 2f.

### 2d: Analyze survivors (parallelized)

Use the Agent tool to analyze survivors **in parallel**. Spawn one subagent per surviving mutant (or per small batch if there are many). Each subagent receives:

- The mutant's `id`, `description`, `diff`, and `original` path.
- Instructions to:
  1. Read the original source file around the mutated lines (parse line numbers from the diff `@@` header) to understand the function being mutated.
  2. Determine which function was mutated, what the mutation changed, and why no existing test caught it.
  3. Return a structured analysis: mutant ID, source file, function name, mutation description, and coverage gap explanation.

Collect all subagent results before proceeding.

### 2e: Write fix tests (sequential)

Skip this step if auto-fix is disabled.

Using the collected analyses from 2d, write fix tests **sequentially** (one file at a time):

1. Group analyzed mutants by source contract.
2. For each group, find existing test files in `test-foundry/` that test the same contract. Prefer adding tests to an existing relevant test file rather than creating a new one.
3. Write focused Foundry tests that would fail if the mutations were applied.
   - For library internal functions, create a harness contract that exposes them.
   - Every test function MUST have a comment: `/// TARGETS MUTANT #<id> in <source-file>`
   - ONLY write tests targeting specific surviving mutants. No bonus tests.
4. Run `forge build --optimize false` then `forge test --optimize false --match-path <test-file>` to confirm they compile and pass against unmodified source. Fix any issues and retry.

### 2f: Write iteration report

Write a markdown report to `<prefix>-<iteration>.md`:

**Header**: Date, iteration number, files tested, mutest command used.

**Summary table**: Total mutants, killed, survived, kill rate. If iteration > 1, note how many previously-surviving mutants were killed.

**Breakdown by source file**: Table with mutant counts per file.

**Surviving mutants** (grouped by source file, then function): For each survivor include mutant ID, mutation type, file/function/line, diff as fenced code block, coverage gap explanation, and fix status.

**Prioritized remaining gaps** (if any unfixed): Rank by severity — Critical (access control, fund transfers, state-changing math), High (business logic, validation), Medium (helpers, view functions, events).

Tell the user the report path.

### 2g: Decide whether to loop

If **loop-until-clean is disabled**, stop here.

If **loop-until-clean is enabled**:
- 0 survivors → stop, all killed.
- Survivors exist and new tests were written → start next iteration (go to Step 2a, no file args).
- Survivors exist but no new tests written → stop to avoid infinite loop. Tell the user which mutants remain.

## Important notes

- **NEVER delete, remove, or clean up `gambit_out/`.** mutest manages this directory and needs it for re-runs.
- All user decisions are collected in Step 1. Do not prompt the user for decisions during the run.
