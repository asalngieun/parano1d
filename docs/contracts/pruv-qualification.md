# Pruv qualification: 15 October 2026

Pruv 0.1.0 was exercised against the existing ABI 3 interpreter and real v2.0.4
node, CLI and miner processes. Compilation, contract funding, native execution,
deadline recovery, pruning and receipt verification passed.

The [machine-readable record](pruv-qualification-20261015.json) contains source,
executable and receipt hashes, proof-bank pins and the confirmed call heights.

## Environment

| Component | Configuration |
| --- | --- |
| Host | Linux x86-64 |
| Rust | 1.96.0 |
| Solver | Z3 5.3.0 |
| Source base | `ba498fe1d835b59a036de915cdf388b8520e4536` |
| Contract runtime | Existing v2.0.4 ABI 3 |
| Live network | Loopback-only namespace, copied stopped databases and a fresh verifier |
| Proof bank | Isolated `isolated-v2-fork-testnet` schedule, v2 at H10 |
| V2 bank digest | `be82f3bec102f03c63715dac9bb4a939cb9aa5b21d013e38402b321fd8a41fa9` |
| CPU affinity | Producer: logical CPUs 0–7; receiver and verifier: logical CPUs 8–11 |

The isolated bank matches the early test schedule. ABI 3 instruction semantics,
wallet authorization and the node's proof construction and verification paths
come from the current source. The Pruv changes add a compiler and developer
tooling; consensus source and matrix pins retain their existing definitions.

## Compiler and model checks

| Check | Result |
| --- | --- |
| Pruv unit and CLI tests, including explicit Z3 tests | 21 passed |
| Existing `noid_tx` unit and contract application tests | 42 passed |
| Source/native differential executions | 202,224 matched |
| Malformed source inputs | 2,048 completed without a parser panic |
| Native/SMT concrete execution cases | 1,920 matched |
| Native opcode coverage | All ten opcodes |
| Resource boundaries | 16 instructions, two persistent and two scratch registers, source/token/depth limits |
| Solver failure handling | Counterexamples, infeasibility, partial answers, errors and timeout rejected |

The SMT cases cover all opcodes, predicate sources, inversions and register
destinations, with boundary operands and both accepted and rejected execution.
Source comparisons cover numeric context operands, flags, checked arithmetic,
instruction order, nested conditions and scratch reuse. CLI tests also cover
duplicate parameters, exact u64 parsing and preservation of existing files.

Postcondition results use Z3's QF_BV semantics and the encoded ABI 3 integer
semantics, with arbitrary incoming u64 state and constrained native context.
The [Pruv guide](pruv.md#postcondition-verification) describes those conditions
and how they connect to lifecycle invariants.

## Live contract lifecycle

[`live_pruv_contract_scenario.py`](../../scripts/live_pruv_contract_scenario.py)
passed with three separately funded programs: the budget example, the example
covering every native opcode, and a counter initialized at `u64::MAX`.

| Stage | Observed result |
| --- | --- |
| Compile and check definitions | `check`, `verify`, `smt` and `build` succeeded |
| Create, watch and fund | Three instances funded through the current CLI at H15 |
| Continue | Four confirmed calls at H16, H17 and H18; successor state matched on both nodes |
| Deadline recovery | All three instances closed at H20, including the overflowing counter |
| Ordinary retention | 43 further blocks produced; every call's block body pruned on both nodes |
| Restart | Receiver reopened its database and selected the exact H63 tip |
| Fresh synchronization | A new verifier reached the same pruned H63 tip |
| Receipts | All seven receipts verified on all three nodes and imported by the fresh wallet |
| Shutdown | All node processes exited successfully |

Eleven negative cases were rejected: wrong authority, wrong creation ID,
excessive payout, wrong recipient, excessive fee, early closing, counter
overflow, stale review, budget underflow, damaged receipt and import with the
wrong opening. Overflow and budget exhaustion produced native program errors;
the recovery branch remained executable at the deadline.

## Documented CLI workflow

[`live_pruv_cli_scenario.py`](../../scripts/live_pruv_cli_scenario.py) then
passed on a fresh copy of the stopped H63 chain using the guide's budget
parameters and ordinary CLI commands.

| Stage | Observed result |
| --- | --- |
| Compile, create, watch and fund | Funded at H64 |
| Review and submit | Exact preview bound to the authorized transaction |
| Wait for confirmation | Payout confirmed at H65; the CLI automatically saved its verified receipt |
| Discover successor | The saved call file resolved to the current instance |
| Deadline recovery | Successor closed at H66; no live instance remained |
| Verify receipts | Payout and closing receipts verified through the CLI on both nodes |
| Negative cases | Wrong reviewed transaction ID and spent predecessor rejected |
| Shutdown | Both node processes exited successfully |

## Reproduction

```sh
cargo fmt -p noid_pruv -- --check
cargo clippy -p noid_pruv --all-targets --no-deps -- -D warnings
PRUV_TEST_Z3=/path/to/z3 cargo test --release --locked -p noid_pruv -p noid_tx -- --include-ignored
```

The [testing guide](../developers/testing.md#pruv-contracts) describes the
isolated binaries, matching proof bank, stopped fixtures and environment
variables used by the live scripts. Each script saves a report, executable
hashes, contract definitions and receipts in its output directory.
