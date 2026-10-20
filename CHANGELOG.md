# Changelog

## Unreleased

### Added

- `get_treasury()` returns the effective platform treasury and `get_pending_treasury()` returns the scheduled treasury update that has not yet taken effect, so clients no longer need to read raw storage.

### Breaking change

- `Enrollment.evidence_hash` and `mark_completed(..., evidence_hash)` now use `Option<BytesN<32>>` instead of `Option<String>`. Evidence is stored as a fixed 32-byte hash to reduce on-chain storage cost and keep verification deterministic.
