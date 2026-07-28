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
parameters.

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
| 4 | RPC wallet + functional test | - |
| 5 | write-up | - |

## Code

https://github.com/jeanpablojp/bitcoin/tree/p2mr-regtest

## Files

NOTES.md -- spec questions and answers, build log.
FINDINGS.md -- ambiguities, vector results, measurements.
FUTURE.md -- what I'm deliberately not doing.

## Spec version

BIP 360 v0.12.x, bitcoin/bips commit
0fdf6ffdbb394a73c80978ae647322ceda8b9337 (2026-07-24). The header
still says 0.12.0 while the changelog is at 0.12.1 -- see FINDINGS.md.

## License

MIT.
