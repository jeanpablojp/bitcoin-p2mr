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
  p2mr_construction.json list them in depth-first order. Looks like a
  sibling of the control-block ordering bug fixed in bips PR #2202 --
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

## Measurements

Stage 4: witness weight and vsize, 2-leaf vs deeper trees, compared
with equivalent P2TR script-path spends.
