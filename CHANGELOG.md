# Changelog

## Unreleased

### Changed

- `revoke_certificate` now panics if `reason` is empty (`revocation reason cannot be empty`) or longer than 64 bytes (`revocation reason exceeds maximum length`). Callers passing an empty or oversize reason must shorten or supply one.

### Breaking change

- `Enrollment.evidence_hash` and `mark_completed(..., evidence_hash)` now use `Option<BytesN<32>>` instead of `Option<String>`. Evidence is stored as a fixed 32-byte hash to reduce on-chain storage cost and keep verification deterministic.
