---
description: Run mutation testing on Solidity files, then optionally analyze and kill surviving mutants.
argument-hint: <sol-file> [sol-file ...]
---

# Mutation Testing

If $ARGUMENTS is empty, tell the user they need to specify at least one Solidity file path and stop.

## Step 0: Verify the test suite passes

Run `forge test` and wait for it to complete. If any tests fail, show the user which tests failed and offer to comment them out so mutation testing can proceed:

1. List the failing test names and their files.
2. Use AskUserQuestion to ask: "These tests are failing. Should I comment them out so we can proceed with mutation testing?" with options:
   - **Yes — comment them out**: Comment out the failing test functions (wrap each in `/* ... */` with a `// COMMENTED OUT FOR MUTEST — was failing before mutation testing` marker). Then re-run `forge test` to confirm the suite now passes. If it still fails, repeat.
   - **No — stop here**: Stop and let the user fix the tests manually. Do not proceed.

Do not proceed until `forge test` exits with 0.

## Step 1: Gather all inputs up front

Use AskUserQuestion to collect everything at once:

1. **Mutest test command override**: Override the command mutest uses to run tests? Default is `forge test --optimize false --threads 1 --root <workerDir>`. If yes, provide the command. If no, leave blank.
2. **Analyze survivors**: After mutest finishes, run `/analyze-survivors` to produce a coverage gap report? (Yes / No)
3. **Kill mutants**: After analysis, run `/kill-mutants` to write tests that kill surviving mutants? (Yes / No)
4. **Build command** (only if killing mutants): Command to build the project. Default: `forge build --optimize false`
5. **Test command** (only if killing mutants): Command to run tests. Default: `forge test --optimize false`

Remember the answers — do not prompt the user again during the run.

## Step 2: Run mutest

If the user provided a mutest test command override, append `-t '<cmd>'` to the invocation.

```
npx @godzillaba/mutest@latest $ARGUMENTS [-t '<cmd>']
```

Use the Bash tool with `run_in_background: true` and wait for it to complete.

After it finishes, print a quick summary: how many mutants were generated and how many survived (read `gambit_out/survivors.json` with `jq`).

If there are no survivors, tell the user all mutants were killed and stop.

## Step 3: Analyze survivors (optional)

If the user chose to analyze survivors, invoke the `/analyze-survivors` skill using the Skill tool.

## Step 4: Kill mutants (optional)

If the user chose to kill mutants, invoke the `/kill-mutants` skill using the Skill tool.

The kill-mutants skill will ask for build/test commands — provide the answers the user already gave in Step 1 so they are not asked again.

## Important notes

- **NEVER delete, remove, or clean up `gambit_out/`.** mutest manages this directory and needs it for re-runs.
- All user decisions are collected in Step 1. Do not prompt the user for decisions during the run.
