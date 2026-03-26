---
description: Start an Arbitrum nitro-testnode and run tasks against it. Trigger on "nitro-testnode", "testnode", "test node", "nitro test node".
argument-hint: [task to run against the testnode]
---

# Nitro Testnode

Start an Arbitrum nitro-testnode and optionally execute a task against it.

## Step 1: Clone or update the repo

Clone to `/tmp/nitro-testnode-claude` if not already present:

```
git clone -b release --recurse-submodules https://github.com/OffchainLabs/nitro-testnode.git /tmp/nitro-testnode-claude
```

If the directory already exists, update it:

```
cd /tmp/nitro-testnode-claude && git fetch origin release && git reset --hard origin/release && git submodule update --init --recursive
```

## Step 2: Determine flags

Based on the user's request, select the appropriate flags for `test-node.bash`. Always include `--init-force` (never use `--init` — it has an interactive prompt that hangs in non-interactive shells).

Available flags:

| Flag | Purpose |
|---|---|
| `--l3node` | Enable L3 chain |
| `--l3-fee-token` | Custom fee token on L3 (requires `--l3node`) |
| `--l3-token-bridge` | L2-L3 token bridge (requires `--l3node`) |
| `--tokenbridge` | L1-L2 token bridge |
| `--l2-anytrust` | AnyTrust L2 |
| `--l2-referenceda` | Reference DA provider |
| `--l2-timeboost` | Timeboost |
| `--l2-tx-filtering` | Transaction filtering |
| `--blockscout` | Block explorer |
| `--pos` | Proof-of-stake L1 |
| `--validate` | WASM validation |
| `--no-simple` | Full node config (separate sequencer/poster/validator) |
| `--batchposters N` | Number of batch posters (0-3) |
| `--redundantsequencers N` | Number of redundant sequencers (0-3) |

If the user's request is ambiguous about which flags to use, ask them with AskUserQuestion before proceeding. If no special features are mentioned, use just `--init-force` with no extra flags.

## Step 3: Start the testnode

Run `test-node.bash` in the background:

```
cd /tmp/nitro-testnode-claude && ./test-node.bash --init-force [selected flags]
```

Use the Bash tool with `run_in_background: true`. Do NOT wait for this command to exit — it runs indefinitely as it keeps the node alive.

## Step 4: Wait for RPC endpoints

Poll each required endpoint until it responds to `eth_chainId`. Check every 5 seconds, with a 20-minute timeout. Use curl:

```
curl -s -X POST -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"eth_chainId","params":[],"id":1}' \
  http://127.0.0.1:<port>
```

Required endpoints:

| Chain | RPC URL | WebSocket |
|---|---|---|
| L1 (geth) | `http://127.0.0.1:8545` | — |
| L2 (Arbitrum) | `http://127.0.0.1:8547` | `ws://127.0.0.1:8548` |
| L3 (only if `--l3node`) | `http://127.0.0.1:3347` | `ws://127.0.0.1:3348` |

Poll L1 first, then L2, then L3 (if applicable). A successful response contains `"result"` in the JSON.

If the timeout is reached, show the user the background process output and stop.

## Step 5: Execute the user's task

Once all endpoints are responding, announce:

```
Nitro testnode is running.

Endpoints:
- L1: http://127.0.0.1:8545
- L2: http://127.0.0.1:8547 (ws: ws://127.0.0.1:8548)
- L3: http://127.0.0.1:3347 (ws: ws://127.0.0.1:3348)  [if --l3node]

Dev account (funded on all chains):
- Address: 0x3f1eae7d46d88f08fc2f8ed27fcb2ab183eb2d0e
- Private key: 0xb6b15c8cb491557369f3c7d2c287b053eb229daa9c22138887752191c9520659
```

If `$ARGUMENTS` contains a task, proceed to execute it. If `$ARGUMENTS` is empty, just report that the testnode is ready and stop.

## Reference

**Helper scripts** — run via `./test-node.bash script <name> [args]` from `/tmp/nitro-testnode-claude`:
- `send-l2` — send tx on L2
- `send-l3` — send tx on L3
- `bridge-funds` — bridge funds between chains
- `print-address` — print contract addresses

**Stopping the testnode**:
```
cd /tmp/nitro-testnode-claude && docker compose down
```
