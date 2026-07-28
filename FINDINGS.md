# FINDINGS

Dated notes. Anything that turns out to be a real spec problem becomes
an issue on bitcoin/bips with a minimal reproduction.

## 2026-07-27

- bip-0360.mediawiki header says Version: 0.12.0 but the top changelog
  entry is 0.12.1 (2026-07-24). Report upstream once stage 0 confirms
  nothing else is off.
- The BIP links P2MR_construction.json; the file on disk is
  p2mr_construction.json (same for the pqc file). Trivial fix upstream.
- Earlier Core PRs, all closed unmerged: bitcoin/bitcoin #33163 (old
  P2QRH form, 2025-08) and #35106-#35109 (2026-04, closed within
  minutes of opening). Nothing open or merged today, and nothing in
  Bitcoin Inquisition. Worth mentioning in the write-up.
- Debates that don't change the target: P2TRv2 (Poinsot/Wuille, Optech
  #412 -- single-key spend roughly 15% cheaper than P2MR) and pubkey
  recovery for EC leaves (starius). conduition's depth-0 ban proposal
  became the anyone-can-spend rule in v0.12.0 (PR #2198).

Found during the full spec read, same day:

- Script Validation says the control byte's low bit "is unused and
  must be 1" but never adds an explicit "Fail if" step, unlike every
  other rule in that section. The footnote implies enforcement (it
  describes the bit catching deserialization bugs "with an immediate
  error"). I implement the strict reading -- reject low bit 0 -- and
  will ask upstream to make the Fail clause explicit.
- The Test Vectors section still links the rust implementation that
  changelog 0.12.1 says was removed (moved to
  jbride/libbitcoinpqc-bindings). Dead link now.
- In Transaction Size and Fees, the 135-byte depth-1 witness example
  labels the merkle path "(empty)" while counting 32 bytes for it. A
  depth-1 path has one 32-byte element; the label is wrong.

From inspecting the test vectors (pinned commit):

- The official vectors are construction-only: script tree in, merkle
  root / scriptPubKey / address / control blocks out. No spending
  transactions or signatures, unlike BIP 341's script-path spend
  vectors. Consensus spend-path testing has no official vectors yet;
  our unit and functional tests fill that gap, and generating proper
  spend vectors could be a useful upstream contribution.
- The vector schema mixes key styles: the two missing-tree error
  vectors use given.script_tree (snake_case), every other vector uses
  given.scriptTree (camelCase). A strict parser trips on this.
- p2mr_single_leaf_script_tree builds a depth-0 output (merkle root ==
  leaf hash). Valid as construction, but spending it is
  anyone-can-spend under v0.12.0 -- worth a comment upstream asking
  whether the vector should carry a warning.

## 2026-07-27, stage 1 closing review

Went back over the whole stage before starting stage 2: re-checked
every rule of the Script Validation section against the code, hunted
for regressions to v0/v1/P2SH/anchor spends, and mutation-tested the
unit suite (break the implementation on purpose, see which tests
notice). Nothing diverges from the pinned spec, block validation off
regtest stays byte-identical to vanilla, no memory-safety problems.
The mutation run did expose four blind spots in the tests --
max-depth path, P2SH-wrapped v2, non-32-byte v2 programs, empty
control block -- covered now. The mempool flag scope question is
written up in NOTES.md.

## 2026-07-27, stage 2: official vectors against the implementation

Harness in src/test/p2mr_vector_tests.cpp, consuming the pinned JSONs.
Result: all 9 vectors in p2mr_construction.json pass every check (leaf
hashes, merkle root, scriptPubKey, bc1z address, control blocks, and a
consensus cross-check walking each control block through
ComputeP2MRMerkleRoot). The pqc file has two real bugs, confirmed with
an independent Python recomputation straight from the spec formulas:

- p2mr_pqc_construction.json, p2mr_three_leaf_complex and
  p2mr_three_leaf_alternative: intermediary.leafHashes[0] and [2] are
  swapped. The values are correct, the order is not; the same trees in
  p2mr_construction.json list them in depth-first order. Looks like the
  same kind of mistake as the control-block ordering bug fixed in bips PR #2202 --
  the fix didn't reach the pqc file's leafHashes.
- p2mr_pqc_construction.json, p2mr_different_version_leaves: the
  control block for the leaf with leafVersion 0xfa starts with byte
  0xc1; by the spec's own control byte rules it must be 0xfb (leaf
  version with the low bit set). The path bytes and the vector's
  merkle root are otherwise consistent with the 0xfa leaf, so the
  published control block cannot satisfy validation as printed.

Both fixed and submitted upstream as bitcoin/bips#2220 (the repo has
issues disabled, so straight to a PR -- same route as the earlier
vector fix #2102). The doc-side problems (stale links, version
header, "(empty)" label) went in bitcoin/bips#2221, which also notes
the schema inconsistencies (scriptTree/script_tree, null vs empty
string for a missing tree) for the maintainers to decide on. The
harness keeps documented exceptions for the two vector bugs that
assert the divergence still reproduces at the pinned commit, so the
upstream fix landing flips the test and tells us to drop them.

## 2026-07-28, stage 4: RPCs and the functional test

Two experimental RPCs (createp2mraddress, signp2mrspend) and
feature_p2mr.py, which builds every witness in Python so the consensus
code is checked against a second implementation of the BIP 341/342
signature message. Review notes worth keeping:

- The RPCs were reachable on every chain before I gated them. On
  mainnet createp2mraddress would hand out a bc1z address that real
  consensus treats as anyone-can-spend, so both RPCs now refuse to run
  outside regtest and are registered as hidden, like the other
  regtest-only RPCs.
- signp2mrspend used to sign whatever script and control block it was
  given. It now recomputes the Merkle root from them and checks it
  against the output being spent, and refuses depth-zero control
  blocks: signing one would suggest the signature protects something
  when it protects nothing.
- Unbounded input was a memory problem, not a theoretical one: a
  400k-leaf request drove the node to 2.7 GB before the cap.
- The Python tree in the test and the C++ tree in the RPC agree up to
  four leaves and diverge for five or more. Neither is wrong (the BIP
  commits to the root, not to a tree shape), and keeping them
  independent is the point of the test, but above four leaves a
  control block from one of them will not reconstruct the other's
  root, so spends built by mixing the two fail.

## Measurements

Measured by feature_p2mr.py on regtest: one input, one P2TR output
(MiniWallet's address), SIGHASH_DEFAULT, a <key> OP_CHECKSIG leaf.
Only the input side differs between the rows, so the deltas are the
witness cost:

| spend | control block | weight | vsize |
|---|---|---|---|
| P2MR, 2-leaf tree | 33 B | 513 | 129 |
| P2MR, 4-leaf tree | 65 B | 545 | 137 |
| P2TR script path, 2-leaf tree | 65 B | 545 | 137 |

The 32-byte saving BIP 360 claims over an equivalent P2TR script path
holds exactly: the control block carries no internal key. Put another
way, a P2MR spend buys one extra level of tree for free relative to
taproot, so the 4-leaf P2MR spend and the 2-leaf taproot spend cost
the same 137 vB.

Not measured here, and worth stating plainly for the P2MR vs P2TRv2
discussion: this compares script paths. A taproot key path spend is a
single 64-byte signature and remains much cheaper than any P2MR spend,
which is the tradeoff the BIP makes for quantum resistance.
