# FUTURE

Possible follow-ups, roughly in order of interest. The first six are
not part of BIP 360, so not part of this project:

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

Done since, both from the in-scope leftovers this file used to carry:
the validation weight budget test (item 8) and the spend-path test
vectors (item 7). See FINDINGS. Nothing in scope is left undone.
