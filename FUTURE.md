# FUTURE

Not part of BIP 360, so not part of this project. Possible follow-ups,
roughly in order of interest:

1. Post-quantum signature opcodes (e.g. OP_CHECKSIG_MLDSA via
   OP_SUCCESSx). This is the companion BIP that BIP 361 lists as TBD;
   a prototype with cost numbers would feed that discussion.
2. A custom signet running the patch.
3. Bitcoin Inquisition adaptation, if the write-up gets interest.
4. BIP 361 phases A/B compressed on regtest, plus commit/reveal
   recovery.
5. Descriptors / PSBT. There is no mr() descriptor BIP.
6. Activation logic. The draft defines no parameters, so there is
   nothing to implement.

Two smaller ones that came out of building this, both inside the
project's scope but left undone:

7. Spend-path test vectors for BIP 360. The official vectors only
   cover construction, so every implementer writes their own spending
   coverage. Contributing vectors upstream would fix that once.
8. A validation weight budget test. It is the one place where P2MR's
   smaller control block changes the BIP 342 arithmetic against
   taproot, and nothing exercises it yet.
