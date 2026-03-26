---
description: Kill surviving mutants in parallel using worktrees — one fixer agent per source file writes and validates tests concurrently.
---

# Kill Mutants

Read surviving mutants from `gambit_out/survivors.json`, spawn one fixer agent per source file in parallel worktrees, write tests that kill the mutants, validate each kill, and merge results back incrementally.

## Survivor Format

Each entry in `gambit_out/survivors.json` has:
- `id`: mutant identifier (NOT globally unique — scoped to `original`)
- `description`: mutation type (e.g. `BinaryOpMutation`)
- `diff`: unified diff showing what changed
- `original`: source file path relative to project root (e.g. `src/bridge/Bridge.sol`)
- `name`: mutant file path relative to `gambit_out/` (e.g. `mutants/1/src/bridge/Bridge.sol`)
- `sourceroot`: project root

The mutant source file lives at: `gambit_out/<original>/mutants/<id>/<original>`
Example: `gambit_out/src/bridge/Bridge.sol/mutants/1/src/bridge/Bridge.sol`

## Progress Tracking

Progress is tracked directly in `gambit_out/survivors.json` by adding fields to each survivor entry:

- `"status"`: one of `"ALREADY KILLED"`, `"KILLED"`, `"NOT KILLED"`, `"TEST BROKEN"`
- `"test"`: the name of the test function written for this mutant (absent for `"ALREADY KILLED"`)

A survivor is **unprocessed** if it has no `"status"` field.

**Only the orchestrator writes to `survivors.json`.** Fixer agents return results; the orchestrator merges them.

**Always use `jq` for reading and manipulating `survivors.json`.** Examples:

```bash
# Count unprocessed survivors
jq '[.[] | select(has("status") | not)] | length' gambit_out/survivors.json

# Get all unprocessed, grouped by original
jq '[.[] | select(has("status") | not)] | group_by(.original) | map({file: .[0].original, count: length})' gambit_out/survivors.json

# Update entries after a fixer agent completes (given a results JSON file)
jq --slurpfile results /tmp/results.json '
  ($results[0] | map({key: (.original + ":" + .id), value: .}) | from_entries) as $lookup |
  [.[] | . as $entry |
    if $lookup[$entry.original + ":" + $entry.id] then
      . + ($lookup[$entry.original + ":" + $entry.id] | {status} + (if .test then {test} else {} end))
    else . end]
' gambit_out/survivors.json > tmp.$$.json && mv tmp.$$.json gambit_out/survivors.json

# Cumulative stats
jq '{total: length, already_killed: [.[] | select(.status == "ALREADY KILLED")] | length, killed: [.[] | select(.status == "KILLED")] | length, not_killed: [.[] | select(.status == "NOT KILLED")] | length, broken: [.[] | select(.status == "TEST BROKEN")] | length, remaining: [.[] | select(has("status") | not)] | length}' gambit_out/survivors.json
```

---

## Orchestrator Workflow

You are the orchestrator. Follow these phases in order.

### Phase 1: Test & Build Commands

Ask the user what commands to use for:
1. **Running tests** (default suggestion: `forge test --optimize false`)
2. **Building** (default suggestion: `forge build --optimize false`)

Do not proceed until you have their answers.

### Phase 2: Load & Group

1. Read `gambit_out/survivors.json` with `jq`.
2. Filter to unprocessed entries (no `"status"` field).
3. Group by `.original` (source file).
4. Print a summary table:

```
Unprocessed survivors by file:
| Source File                          | Mutants |
|--------------------------------------|---------|
| src/bridge/Bridge.sol                |      12 |
| src/bridge/SequencerInbox.sol        |      45 |
| src/rollup/RollupCore.sol            |       8 |
| TOTAL                                |      65 |
```

5. If no unprocessed survivors remain, print "All survivors have been processed", show cumulative stats, and stop.

### Phase 3: Spawn Fixer Agents

Spawn **one Agent per source file**, all in the same message (parallel). Use these parameters for each:

```
Agent(
  prompt: <fixer agent instructions + data — see template below>,
  isolation: "worktree",
  run_in_background: true
)
```

For each agent, construct the prompt by copying the **Fixer Agent Instructions** section below verbatim, then appending:

1. `BUILD_COMMAND: <the user's build command from Phase 1>`
2. `TEST_COMMAND: <the user's test command from Phase 1>`
3. `SOURCE_FILE: <the original path, e.g. src/bridge/Bridge.sol>`
4. `SURVIVORS:` followed by the JSON array of that file's unprocessed survivors (from jq output)

Spawn ALL agents in the same message. Do not wait for one to finish before spawning the next.

### Phase 4: Merge As They Complete

As each background agent completes (you'll be notified):

1. **Parse results**: The agent's final message contains a JSON array of result objects. Extract it.
2. **Merge branch**: The agent worked in a worktree on its own branch. Merge that branch into the current branch:
   ```bash
   git merge <branch-name> --no-edit --no-gpg-sign
   ```
   If the merge has conflicts, try to resolve them (they should be trivial — agents edit different files). If auto-resolution fails, abort the merge, log a warning, and note the branch name for manual inspection.
3. **Update survivors.json**: Write the agent's results into `survivors.json` using `jq`. For each result entry, update the matching survivor (by `id` + `original`) with the `status` and `test` fields.
4. **Print progress**: Show which file completed, how many mutants were processed, and the breakdown (killed/not killed/already killed/broken). Then show cumulative stats across all completed agents so far.

### Phase 5: Final Report

After ALL agents have completed and been merged:

1. Print a full summary table:

```
| Source File                   | Total | Already Killed | Killed | Not Killed | Broken |
|-------------------------------|-------|----------------|--------|------------|--------|
| src/bridge/Bridge.sol         |    12 |              2 |      8 |          1 |      1 |
| src/bridge/SequencerInbox.sol |    45 |              5 |     35 |          3 |      2 |
| TOTAL                         |    57 |              7 |     43 |          4 |      3 |
```

2. Print cumulative stats from the full `survivors.json` (including previously processed entries).
3. List any agents that failed or branches that couldn't be merged automatically.

---

## Fixer Agent Instructions

**Copy everything between the `===BEGIN===` and `===END===` markers (inclusive of content, exclusive of markers) into each agent's prompt.**

===BEGIN===

You are a fixer agent. You will kill surviving mutants for a single Solidity source file by writing tests that detect each mutation.

## Setup

First, symlink `node_modules` and verify the build:

```bash
ln -s /workspace/node_modules ./node_modules
<BUILD_COMMAND>
```

If the build fails, check that the symlink is correct and retry. If it still fails, check if git submodules need updating:

```bash
git submodule update --init --recursive
```

## Inputs

You will receive:
- `BUILD_COMMAND`: the command to build the project (e.g. `forge build --optimize false`)
- `TEST_COMMAND`: the command to run tests (e.g. `forge test --optimize false`)
- `SOURCE_FILE`: the source file you are responsible for (e.g. `src/bridge/Bridge.sol`)
- `SURVIVORS`: a JSON array of survivor entries for your file

## Mutant File Paths

Mutant source files live at absolute paths in the main workspace:
`/workspace/gambit_out/<original>/mutants/<id>/<original>`

Example: `/workspace/gambit_out/src/bridge/Bridge.sol/mutants/1/src/bridge/Bridge.sol`

To apply a mutant: `cp /workspace/gambit_out/<original>/mutants/<id>/<original> <original>`
To restore: `git checkout -- <original>`

**Always restore the original after each validation step, even on errors.**

## Step 1: Pre-check (Already Killed?)

For each survivor, check if existing tests already kill it:

1. Apply mutant: `cp /workspace/gambit_out/<original>/mutants/<id>/<original> <original>`
2. Run the full test suite: `<TEST_COMMAND>`
3. If any test fails — mark this survivor as `ALREADY KILLED` in your results and skip it.
4. Restore original: `git checkout -- <original>`

Only survivors that pass all existing tests proceed to Step 2.

## Step 2: Write Tests

For survivors that are NOT already killed:

1. Read each survivor's `diff` to understand the mutation.
2. Read the original source around the mutated lines.
3. Find the natural home for the test — look for an existing test file that already tests the same contract. If none exists, create one following existing test conventions in the project.
4. Write focused test functions that exercise the exact code path each mutant changes. The test should pass against original code but fail with the mutation applied.
5. **Tests must blend in with existing tests.** No references to mutants or mutation testing in test names. Name and style tests exactly like surrounding tests.
6. Each test function MUST have a comment above it: `// KILLS MUTANT #<id> in <source-file>` — this is the only mutation-related marker allowed.

**Internal batching**: If you have more than ~15 survivors, process them in batches of ~15. Write tests for a batch, validate, then move to the next batch.

After writing tests, run `<TEST_COMMAND>` to make sure all new tests compile and pass against the unmodified source.

## Step 3: Validate Each Kill

For EACH survivor you wrote a test for, one at a time:

1. **Apply mutant:**
   ```
   cp /workspace/gambit_out/<original>/mutants/<id>/<original> <original>
   ```

2. **Run targeted test:**
   ```
   <TEST_COMMAND> --match-test <test_function_name>
   ```

3. **Check: the test MUST FAIL.** If it passes, this test does not kill the mutant → `NOT KILLED`.

4. **Restore original:**
   ```
   git checkout -- <original>
   ```

5. **Run targeted test again:**
   ```
   <TEST_COMMAND> --match-test <test_function_name>
   ```

6. **Check: the test MUST PASS.** If it fails, the test is broken → `TEST BROKEN`.

7. If `NOT KILLED` or `TEST BROKEN`, attempt to fix the test and re-validate (up to 2 fix attempts). If still failing, record the status and move on.

## Step 4: Commit & Return Results

1. **Commit all changes** (new/modified test files only) with message: `test: kill mutants in <source-file>`. Use `--no-gpg-sign`.

2. **Return results** as a JSON array in your final message. The JSON MUST be the last thing in your message, wrapped in a markdown code block tagged `json-results`:

````
```json-results
[
  {"id": "1", "original": "src/bridge/Bridge.sol", "status": "KILLED", "test": "test_initialize_setsActiveOutbox"},
  {"id": "2", "original": "src/bridge/Bridge.sol", "status": "ALREADY KILLED"},
  {"id": "3", "original": "src/bridge/Bridge.sol", "status": "NOT KILLED", "test": "test_enqueueDelayedMessage_reverts"},
  {"id": "4", "original": "src/bridge/Bridge.sol", "status": "TEST BROKEN", "test": "test_setOutbox_onlyRollup"}
]
```
````

**Every survivor you received MUST appear in the results array with a status.**

## Important Rules

- Always restore the original file with `git checkout -- <original>` after each validation, even on error.
- Never leave a mutant applied to the source tree.
- Run only the targeted test during validation (`--match-test`), not the full suite.
- If `forge build` fails after writing tests, fix compilation errors before starting validation.
- Do NOT modify `survivors.json` — return results and the orchestrator will update it.
- Do NOT modify any source files other than test files.

===END===

---

## Worktree Notes

- `gambit_out/` is gitignored — agents use absolute paths (`/workspace/gambit_out/...`) to read mutant files from the main workspace.
- `node_modules/` is gitignored — agents symlink from `/workspace/node_modules`.
- Merge conflicts should be minimal since each agent edits different test files (mostly appending).
- Each worktree builds its own forge cache automatically.
- Git submodules are inherited by worktrees; agents run `git submodule update --init` if needed.

## Error Handling

- **Agent crashes**: Those mutants remain unprocessed for a future run. The worktree is cleaned up automatically by the Agent tool.
- **Merge conflict**: Attempt auto-resolution. If it fails, abort merge, log warning, keep branch for manual inspection. The mutants are still unprocessed.
- **Build fails**: Agent retries with symlink check. If still broken, agent reports error and its mutants remain unprocessed.

## Important Rules

- Only the orchestrator writes to `survivors.json`.
- Always merge agent branches with `--no-gpg-sign`.
- After merging each agent, verify that tests still pass before merging the next agent (optional — skip if agents edit different files).
- Worktrees are cleaned up automatically by the Agent tool after the agent completes.
- Do NOT skip validation. Every test must be proven to kill its mutant.
