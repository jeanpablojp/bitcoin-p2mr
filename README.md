# bitcoin-p2mr

**What is this?** A working prototype of Bitcoin's proposed
quantum-resistant address type. BIP 360 is a draft proposal that
defines a new kind of Bitcoin output designed to survive quantum
computers; this project implements its rules inside Bitcoin Core and checks the
result against the official test vectors, as a way of testing whether
the spec is complete and unambiguous. The goal is to learn the spec
deeply and report anything unclear back upstream -- not to ship
production software.

The code lives in a Bitcoin Core fork (linked below). This repository
is the project journal: plan, spec questions, findings, and results.

In one line: BIP 360 (Pay-to-Merkle-Root / P2MR) consensus rules
implemented in Bitcoin Core, checked against the official test
vectors. Regtest only: BIP 360 is a draft and has no activation
parameters, so block validation applies the rules on regtest and
nowhere else. Note that the mempool side is not chain-gated; the
trade-off is written up in NOTES.md.

Independent, educational implementation. Not affiliated with Bitcoin
Core or the BIP 360 authors. Not for mainnet use.

## Why quantum? (plain-language summary)

Bitcoin coins are protected by digital signatures based on
elliptic-curve cryptography. A large enough quantum computer running
Shor's algorithm could take a *public* key and compute the matching
*private* key -- which means it could spend anyone's coins whose
public key is visible on the blockchain. Today's addresses hide the
public key only until you spend; some older or reused addresses
expose it permanently.

BIP 360 (Pay-to-Merkle-Root, P2MR) is a proposed new output type that
prepares Bitcoin for that scenario. It is like taproot with the
vulnerable part removed: instead of committing to an elliptic-curve
public key, an output commits only to the merkle root of a tree of
spending scripts. No key, nothing for a quantum computer to attack at
the output level, and the script leaves can carry post-quantum
signature schemes as they become available.

## Status

| Stage | | |
|---|---|---|
| 0 | vanilla build + spec reading | done |
| 1 | consensus rules | done |
| 2 | official test vectors | done |
| 3 | address / solver | done |
| 4 | RPC wallet + functional test | done |
| 5 | write-up | in progress |

## What is implemented

Nine commits on top of Bitcoin Core master, all of them prefixed
`p2mr:`. Roughly:

| part | lines |
|---|---|
| consensus (script interpreter, error codes) | ~96 |
| policy and block script flags | ~33 |
| address recognition (solver, key_io, RPC surfaces) | ~59 |
| experimental regtest RPCs | 329 |
| unit tests | ~540 |
| functional test | 492 |

The consensus change is small because BIP 360 is deliberately taproot
with the key path removed: the witness v2 branch reuses the existing
tapscript execution, the BIP 342 signature message and the Merkle path
walk. What differs:
the witness program is compared against the computed root with no key
tweak, the control block is 1 + 32*m bytes instead of 33 + 32*m,
depth-zero trees succeed without executing anything, and there is no
key path (so a two-element witness ending in an annex, which taproot
accepts, is invalid here). The low bit of the control byte must be
set, which needs its own error code.

One thing that surprised me and is worth repeating to anyone
implementing this: patching the interpreter is not enough.
`PrecomputedTransactionData::Init()` decides whether to precompute the
BIP 341 sighash midstate by pattern-matching spent outputs, and it
does not recognise witness v2. Until that is fixed, everything looks
fine right up to the moment you verify a real signature.

## Results

- All nine vectors in the official `p2mr_construction.json` are
  satisfied: the seven construction cases match byte for byte (leaf
  hashes, Merkle root, scriptPubKey, bech32m address, control blocks),
  each control block is walked back to the root through the consensus
  code. The remaining two are error vectors: the harness checks they
  declare an error and skips them (one of the two also publishes a
  scriptPubKey alongside its error, which the harness deliberately
  ignores).
- Three vectors in `p2mr_pqc_construction.json` carry two bugs
  between them. Fix submitted as
  [bitcoin/bips#2220](https://github.com/bitcoin/bips/pull/2220).
  Documentation problems (dead links, a stale version header, a
  mislabelled size example) were submitted as
  [bitcoin/bips#2221](https://github.com/bitcoin/bips/pull/2221).
- The functional test spends P2MR outputs end to end on regtest and
  rejects an invalid spend inside a hand-built block, which is the
  point: these are consensus rules, not mempool policy.
- Witness weight was measured against an equivalent taproot script
  path spend; the numbers are in FINDINGS.md. Checking the BIP's
  "always 32 bytes smaller" claim at every depth turned up an
  exception at depth 7 (34 bytes, a compact size boundary); fix
  submitted as
  [bitcoin/bips#2223](https://github.com/bitcoin/bips/pull/2223).

## Running it

The code is in the fork, not in this repository:

```
git clone -b p2mr-regtest https://github.com/jeanpablojp/bitcoin.git
cd bitcoin
cmake -B build && cmake --build build -j 8
ctest --test-dir build                       # includes the vector tests
build/test/functional/feature_p2mr.py        # end-to-end on regtest
```

The official vector JSONs are vendored into the fork at the pinned
spec commit, so the vector tests run from a plain build.

Two RPCs are available on regtest for building and spending P2MR
outputs by hand: `createp2mraddress` and `signp2mrspend`. They are
hidden RPCs and refuse to run on any other chain, because a witness v2
output is anyone-can-spend everywhere BIP 360 is not active.

## Code

https://github.com/jeanpablojp/bitcoin/tree/p2mr-regtest

## Files

NOTES.md -- spec questions and answers, build log.
FINDINGS.md -- ambiguities, vector results, measurements.
FUTURE.md -- follow-ups: out-of-scope work, plus two in-scope gaps
left undone.

## Spec version

BIP 360 v0.12.x, bitcoin/bips commit
0fdf6ffdbb394a73c80978ae647322ceda8b9337 (2026-07-24). The header
still says 0.12.0 while the changelog is at 0.12.1 -- see FINDINGS.md.

## License

MIT.
