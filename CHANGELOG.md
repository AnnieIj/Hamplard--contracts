# Changelog

## Unreleased

### Breaking changes
- `Enrollment.evidence_hash` and `mark_completed(..., evidence_hash)` now use `Option<BytesN<32>>` instead of `Option<String>`. Evidence is stored as a fixed 32-byte hash to reduce on-chain storage cost and keep verification deterministic.
- `set_arbitration_fee_config` now takes a `token` argument, and `get_arbitration_fee_config` is keyed by token. Arbitration fees are stored under `DataKey2::ArbitrationFeeConfig(token)`.
- Contract failures now return stable `Error` codes instead of string panic messages. Clients should match on `HostError: Error(Contract, #N)` and the code table in `README.md`.

### Added
- Per-token arbitration fee reserve via `DataKey2::ArbitrationFeesHeld(token)` and `get_arbitration_fees_held()`.
- Escalation credits the reserve; resolution debits it before payout.
- `withdraw_tokens()` now excludes reserved arbitration fees from the withdrawable contract balance.
- Documentation for the actual revenue split, co-instructor limitation, security model, error codes, and current test suite.

### Changed
- Enrollment counter updates now use shared `inc_total_active` / `dec_total_active` helpers.
