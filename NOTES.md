# NOTES

Spec questions for stage 0, answered 2026-07-27 against
bip-0360.mediawiki at commit
0fdf6ffdbb394a73c80978ae647322ceda8b9337 (header v0.12.0, changelog
0.12.1). Sections cited by name.

Q1. Control block first byte. Leaf version is
v = c[0] & 0xfe (Script Validation); the low bit "is unused and must
be 1", and Design adds "The parity bit of the control byte is always
1, since P2MR does not have a key path spend". A footnote explains
why: it catches implementations that read the leaf version as c[0]
instead of c[0] & 0xfe. Caveat: the validation steps never state
"Fail if the low bit is 0" -- every other rule in that section has an
explicit Fail clause. The footnote implies enforcement, so I take the
strict reading: reject. See FINDINGS. Needs a negative test.

Q2. Annex. Identical to BIP 341. Script Validation: fail if
fewer than two witness elements; fail if exactly two and the last
starts with 0x50; with three or more, a last element starting with
0x50 is the annex -- removed from the stack, covered by the
signature, contributes to weight, otherwise ignored.

Q3. Tree hash tags. TapLeaf and TapBranch, both reused
unchanged. k0 = hash_TapLeaf(v || compact_size(size of s) || s), then
hash_TapBranch over each pair in lexicographic order -- the BIP 341
construction verbatim (Design + Script Validation).

Q4. Unknown leaf versions. Unencumbered, as in taproot.
"This implies that for the future leaf versions (non-0xC0) the
execution must succeed" (Script Validation); Backward Compatibility
repeats that new leaf versions are the upgrade path.

Q5. Depth 0. Official since v0.12.0. Order matters: the
merkle check q == r runs first, then "If m = 0, succeed immediately"
-- execution is skipped. So a depth-0 spend still has to reveal the
preimage, a script s whose TapLeaf hash equals the program.
"Anyone-can-spend" means anyone who knows s can take the coins
without satisfying it, and s goes public the moment a spend hits the
mempool. Smallest protected tree is 2 leaves; our RPCs must refuse to
build depth-0 outputs. Functional test case 3 builds the witness
accordingly: revealed script + 1-byte control block.

Q6. Signature message. Per the spec, "exactly the same procedure as
defined in BIP342 Common Signature Message" (own section). So
SignatureHashSchnorr() as-is: tapscript extension with tapleaf_hash,
key_version 0, codesep_pos; spend_type = 2*ext_flag + annex_present.
Zero new sighash code. Stage 2 vectors are the proof.

Q7. Standardness / policy. The spec is silent for upgraded
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

- Merkle root computation is shared with taproot by design of the BIP
  itself (same TapLeaf/TapBranch tags, same ordering). Rather than
  duplicating the ~10-line path walk (two copies of consensus code
  that must never drift) or exposing a base-size integer parameter
  (any value would typecheck), the implementation keeps one private
  helper and exposes two named entry points, ComputeTaprootMerkleRoot
  and ComputeP2MRMerkleRoot. The public API only offers the two valid
  control block shapes.
- Policy mirrors taproot for v2 spends in IsWitnessStandard(): annex
  is non-standard, tapscript initial stack items capped at 80 bytes.
  Without this, P2MR spends would have had no standardness limits at
  all -- the flag being in the mandatory set removes the
  "discouraged upgradable witness program" barrier.
- Flag scope: SCRIPT_VERIFY_P2MR sits in the mandatory policy flags
  on every chain, while block validation only applies it on regtest.
  On mainnet this fork's mempool would diverge from the network (it
  would relay P2MR-shaped v2 spends that real consensus treats as
  anyone-can-spend). I'm keeping it and documenting it instead of
  coding around it: chain-dependent standard flags are not idiomatic
  in Core, this fork must not run off regtest anyway, and there is no
  ban/DoS vector -- block validation off regtest stays byte-identical
  to vanilla.
  Stage 3 update: the solver now types v2/32 outputs, which drops the
  old "witness program is undefined" input-standardness rejection.
  That gate was what still kept this fork's mempool vanilla-ish off
  regtest. From here the fork's mempool applies BIP 360 to v2/32
  spends on every chain: valid ones relay, invalid ones are rejected
  under the mandatory flag (a consensus-classified rejection of a
  transaction mainnet consensus would accept). Needed for the stage 4
  functional tests. Same trade-off and same reasoning as above, now
  with one less safety net, so it bears repeating: this fork is for
  regtest, full stop.
- Depth-0 policy is stricter than consensus on purpose: the stack
  item size cap applies even though consensus skips execution at
  m = 0. Stricter policy is safe, and depth-0 outputs are
  anyone-can-spend anyway.
- Known coverage gaps deferred to stage 4's feature_p2mr.py:
  validation weight budget exhaustion, IsWitnessStandard() rules, and
  an independent (Python) sighash check. The stage 2 vectors give the
  construction side an implementation-independent oracle.
  Stage 4 closed the last two: the functional test builds every
  witness from its own BIP 341/342 signature message, and covers the
  80-byte stack item limit on both sides of the boundary. Budget
  exhaustion is still untested; it needs a signature-heavy leaf and is
  the one place where P2MR's smaller control block changes the
  arithmetic against taproot.
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

## Implementation notes (stage 4)

- The RPCs take a private key per call and store nothing. That side-
  steps the key persistence question the plan left open, and suits a
  tool whose only job is to exercise regtest.
- createp2mraddress refuses single-leaf trees. Under BIP 360 a
  depth-zero tree succeeds without executing anything, so an address
  built from one leaf would be anyone-can-spend; signp2mrspend refuses
  depth-zero control blocks for the same reason.
- Leaves are wrapped as <key> OP_CHECKSIG when they are 32 bytes and
  used as raw tapscript otherwise. Any leaf version other than 0xc0
  has to be assembled by hand; the tool only builds tapscript leaves.
- The functional test deliberately reimplements the script tree in
  Python. It reproduces an official vector's Merkle root, scriptPubKey
  and control blocks, which ties the spec, the Python model and the
  consensus code together in one place.

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
