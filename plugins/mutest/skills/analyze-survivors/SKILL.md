---
description: Analyze surviving mutants from gambit_out/survivors.json and produce a structured report of implied test coverage gaps.
---

# Analyze Survivors

Analyze surviving mutants from `gambit_out/survivors.json` and produce a structured, human-readable report of implied test coverage gaps.

## Survivor Format

Each entry in `gambit_out/survivors.json` has:
- `id`: mutant identifier (scoped to `original`, NOT globally unique)
- `description`: mutation type (e.g. `BinaryOpMutation`, `DeleteExpressionMutation`, `IfStatementMutation`)
- `diff`: unified diff showing what changed
- `original`: source file path relative to project root (e.g. `src/bridge/Bridge.sol`)
- `name`: mutant file path relative to `gambit_out/`
- `sourceroot`: project root
- `status` (optional): one of `"ALREADY KILLED"`, `"KILLED"`, `"NOT KILLED"`, `"TEST BROKEN"` — set by kill-mutants
- `test` (optional): name of the test that killed it

## Scope

Analyze all survivors that represent real coverage gaps. This means all entries EXCEPT those with `"status": "KILLED"` or `"status": "ALREADY KILLED"`. Include:
- Unprocessed entries (no `"status"` field) — not yet attempted
- `"NOT KILLED"` entries — attempted but no test could kill them
- `"TEST BROKEN"` entries — test was written but broken

## Architecture

You are an **orchestrator**. You will spawn **one subagent per source file** to analyze that file's survivors in parallel. You then collect results and render the final report.

### Phase 1: Load and Summarize

1. Read `gambit_out/survivors.json` using `jq`.
2. Print a status summary:
   ```
   Survivors: X total
     Killed: N
     Already Killed: N
     Not Killed: N
     Test Broken: N
     Unprocessed: N
     -> Analyzing: N (unprocessed + not killed + broken)
   ```
3. Filter to analyzable entries (everything except KILLED and ALREADY KILLED).
4. Group by `.original` (source file).
5. Print the file breakdown:
   ```
   Files to analyze:
     src/bridge/SequencerInbox.sol — 200 survivors
     src/bridge/GasRefunder.sol — 120 survivors
     ...
   ```

### Phase 2: Spawn File Analysis Agents

For each source file that has survivors to analyze, spawn a subagent using the Agent tool. **Spawn all agents in parallel** (multiple Agent tool calls in one message).

Each subagent prompt MUST include:

1. The full text of the **File Analysis Agent Instructions** below.
2. The source file path.
3. The JSON array of survivor entries for that file (serialize the array from jq output).

### Phase 3: Collect and Render Report

After all subagents complete, collect their structured JSON results and render the final markdown report.

#### Report Format

```markdown
# Coverage Gap Analysis

Generated: <date>
Scope: N survivors across M files (excluding N killed)

## src/bridge/SequencerInbox.sol (200 survivors, 15 gaps)

### 1. Batch poster authorization is not verified
- **Survivors**: #12, #14, #15
- **Functions**: `addSequencerL2BatchFromOrigin`, `addSequencerL2BatchFromBlobs`
- **Mutation types**: IfStatementMutation, DeleteExpressionMutation
- **Impact**: HIGH
- **Description**: Flipping or deleting the `isBatchPoster` check does not cause any test to fail. No test verifies that unauthorized callers are rejected from submitting batches.

### 2. ...

---

## src/bridge/GasRefunder.sol (120 survivors, 8 gaps)

...

---

## Summary

| File | Survivors | Gaps | HIGH | MEDIUM | LOW |
|------|-----------|------|------|--------|-----|
| SequencerInbox.sol | 200 | 15 | 3 | 8 | 4 |
| GasRefunder.sol | 120 | 8 | 1 | 5 | 2 |
| ... | | | | | |
| **Total** | **N** | **N** | **N** | **N** | **N** |
```

After rendering the report, write it to `gambit_out/coverage-gaps.md` so it persists.

---

## File Analysis Agent Instructions

Copy the instructions below verbatim into each subagent prompt, followed by the file path and survivor data.

```
You are a file analysis agent. Your job is to analyze surviving mutants for a single Solidity source file and identify the test coverage gaps they imply.

You MUST return your results as a single JSON object. Do NOT write or modify any files. You are read-only.

## Input

You will receive:
1. A source file path (relative to project root)
2. A JSON array of survivor entries for that file

## Process

1. Read the source file to understand the contract structure — functions, modifiers, state variables, access control.
2. For each survivor, parse its `diff` to determine:
   - Which function is affected (identify the function name from surrounding context in the diff or by reading the source at the relevant line)
   - What the mutation does in plain English (e.g. "replaces subtraction with modulo", "deletes assignment to state variable", "forces if-condition to always be true")
   - What behavior would go undetected if this mutant survived
3. Cluster survivors into conceptual coverage gaps. Mutants belong to the same gap when they imply the same missing test. Grouping heuristics:
   - Multiple mutations on the same state assignment in the same function -> one gap about that state update
   - Multiple operator mutations on the same arithmetic expression -> one gap about that computation
   - Mutations on an access control check (onlyOwner, require(msg.sender == ...), etc.) -> one gap about authorization
   - Mutations on different lines but in the same logical block (e.g. a multi-step state update) -> one gap about that block
   - Do NOT just group by mutation type — group by the semantic behavior that is untested
4. For each gap, classify impact:
   - HIGH: Access control bypass, fund loss/theft, authorization skip, critical state corruption
   - MEDIUM: State not persisted correctly, wrong computation result, missing input validation, broken invariant
   - LOW: Event emission only, redundant/defensive check, unreachable code path, cosmetic

## Output

Return EXACTLY one JSON object (no markdown fences, no commentary before or after):

{
  "file": "<source file path>",
  "total_survivors": <number>,
  "gaps": [
    {
      "title": "<short plain-English description of the coverage gap>",
      "survivor_ids": ["<id1>", "<id2>"],
      "functions": ["<function_name1>", "<function_name2>"],
      "mutation_types": ["<type1>", "<type2>"],
      "impact": "HIGH|MEDIUM|LOW",
      "description": "<2-3 sentence explanation of what is not tested and why it matters>"
    }
  ]
}

IMPORTANT:
- The title should be a plain-English statement of what is NOT tested, not a description of the mutation. Good: "setDisallower does not verify the address is persisted". Bad: "DeleteExpressionMutation on line 94".
- Keep descriptions actionable — a developer should understand what test to write from reading the gap.
- Every survivor MUST appear in exactly one gap. Do not drop any.
- Return ONLY the JSON object. No other text.
```

## Important Rules

- Do NOT modify any files in the source tree or test files.
- Do NOT modify `gambit_out/survivors.json`.
- The only file you write is `gambit_out/coverage-gaps.md` (the final report).
- If a subagent fails or returns malformed output, note it in the report and continue with other files.
- If there are no analyzable survivors, report "All survivors have been killed" and exit.
