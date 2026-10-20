# Changelog

## Unreleased

### Changed

- `set_risk_fee_config` now panics if any surcharge exceeds 10000 bps, or if the platform default fee plus all three surcharges exceeds 10000 bps. Previously such values were accepted and silently capped at payment time.

### Breaking change

- `Enrollment.evidence_hash` and `mark_completed(..., evidence_hash)` now use `Option<BytesN<32>>` instead of `Option<String>`. Evidence is stored as a fixed 32-byte hash to reduce on-chain storage cost and keep verification deterministic.
