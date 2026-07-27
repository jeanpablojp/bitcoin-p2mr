# NOTES

Spec questions for stage 0, answered 2026-07-27 against
bip-0360.mediawiki at commit
0fdf6ffdbb394a73c80978ae647322ceda8b9337 (header v0.12.0, changelog
0.12.1). Sections cited by name.

Q1. Control block first byte. Answered. Leaf version is
v = c[0] & 0xfe (Script Validation); the low bit "is unused and must
be 1", and Design adds "The parity bit of the control byte is always
1, since P2MR does not have a key path spend". A footnote explains
why: it catches implementations that read the leaf version as c[0]
instead of c[0] & 0xfe. Caveat: the validation steps never state
"Fail if the low bit is 0" -- every other rule in that section has an
explicit Fail clause. The footnote implies enforcement, so I take the
strict reading: reject. See FINDINGS. Needs a negative test.

Q2. Annex. Answered: identical to BIP 341. Script Validation: fail if
fewer than two witness elements; fail if exactly two and the last
starts with 0x50; with three or more, a last element starting with
0x50 is the annex -- removed from the stack, covered by the
signature, contributes to weight, otherwise ignored.

Q3. Tree hash tags. Answered: TapLeaf and TapBranch, both reused
unchanged. k0 = hash_TapLeaf(v || compact_size(size of s) || s), then
hash_TapBranch over each pair in lexicographic order -- the BIP 341
construction verbatim (Design + Script Validation).

Q4. Unknown leaf versions. Answered: unencumbered, as in taproot.
"This implies that for the future leaf versions (non-0xC0) the
execution must succeed" (Script Validation); Backward Compatibility
repeats that new leaf versions are the upgrade path.

Q5. Depth 0. Answered: official since v0.12.0. Order matters: the
merkle check q == r runs first, then "If m = 0, succeed immediately"
-- execution is skipped. So a depth-0 spend still has to reveal the
preimage, a script s whose TapLeaf hash equals the program.
"Anyone-can-spend" means anyone who knows s can take the coins
without satisfying it, and s goes public the moment a spend hits the
mempool. Smallest protected tree is 2 leaves; our RPCs must refuse to
build depth-0 outputs. Functional test case 3 builds the witness
accordingly: revealed script + 1-byte control block.

Q6. Signature message. Answered: "exactly the same procedure as
defined in BIP342 Common Signature Message" (own section). So
SignatureHashSchnorr() as-is: tapscript extension with tapleaf_hash,
key_version 0, codesep_pos; spend_type = 2*ext_flag + annex_present.
Zero new sighash code. Stage 2 vectors are the proof.

Q7. Standardness / policy. Answered: the spec is silent for upgraded
nodes. The only related text is Compatibility with BIP 141: old nodes
treat v2 as anyone-can-spend and "generally will not relay or mine
such transactions" (Core's DISCOURAGE_UPGRADABLE_WITNESS_PROGRAM
behavior). Policy for P2MR-aware nodes is our call: start with
-acceptnonstdtxn=1, tighten at the end of stage 1.

Q8. Vectors. bip-0360/ref-impl/common/tests/data/, files
p2mr_construction.json (9 vectors) and p2mr_pqc_construction.json
(7 vectors); lowercase on disk, the BIP links them with uppercase (see
FINDINGS). Ref-impl is python/p2mr.py since PR #2202; the rust/js ones
were moved out of the bips repo. Pinned commit above.

Layout: {version: 1, test_vectors: [...]}. Each vector has id,
objective, given.scriptTree (nested tree; leaves carry script/asm/
leafVersion), intermediary.leafHashes + merkleRoot, and either
expected.{scriptPubKey, bip350Address, scriptPathControlBlocks} or
expected.error. Control blocks confirm the Q1 rule (leaf 0xc0 shows up
as 0xc1). They are construction-only vectors: no spending
transactions or signatures, unlike BIP 341's script-path spend
vectors. Spend-path testing is entirely on us (see FINDINGS).

## Implementation notes (stage 1)

- The consensus delta is not limited to VerifyWitnessProgram():
  PrecomputedTransactionData::Init() decides whether to precompute the
  BIP 341 sighash midstate by pattern-matching spent outputs (34 bytes
  starting with OP_1). A P2MR output (OP_2) wasn't matched, so
  m_bip341_taproot_ready stayed false and every signed P2MR spend
  failed sighash computation. Caught by the first unit test; fixed by
  matching OP_2/34-byte outputs the same way. Worth flagging in the
  write-up: any implementation that only patches the interpreter will
  appear to work until it verifies a real signature.
- Two vanilla functional tests assumed v2/32-byte programs are
  unencumbered (feature_taproot "applic" cases, p2p_segwit
  test_segwit_versions). Adapted them following the upstream v1
  precedent (33-byte program or skip). Expected collateral of any new
  witness version taking effect.

## Build log (stage 0)

Done 2026-07-27.

- environment: macOS 26.5.1, Intel i9-9980HK (8c/16t), 64 GB RAM,
  Apple clang 21.0.0, cmake 4.4.0 (brew), Python 3.12.8
- cmake -B build: first run failed, Cap'n Proto missing (multiprocess
  is on by default now); brew install capnp fixed it. ~55 s cold,
  ~16 s warm.
- cmake --build build -j16: 8 min 47 s, success (warnings only).
- ctest --test-dir build -j16: 61 s, all passed; script_assets_tests
  skipped (needs an external data file, expected).
- feature_taproot.py: 2 min 43 s, passed.
- regtest smoke test: OK. bitcoind -regtest -listen=0, createwallet,
  getnewaddress, generatetoaddress 101, balance 50 BTC, clean stop.
  Gotcha for scripts: with -listen on, bitcoind also binds the tor
  target on p2p port + 1, so never pick rpcport = port + 1.
