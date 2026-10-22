# Changelog

## Unreleased

<<<<<<< HEAD
<<<<<<< HEAD
<<<<<<< HEAD
### Storage layout change

- `Course` gains two fields, `total_refunded: i128` and `refund_count: u32`, updated by both refund paths (`process_refund` and `archive_course`) so admins can read per-course refund rates on-chain via `get_course`. `Course` is a `#[contracttype]` struct, so courses stored before this change lack the new keys and will not deserialize against the new type; redeploy fresh, or migrate existing `Course` / `ArchivedCourse` entries before upgrading a live contract.
=======
### Added

- `get_treasury()` returns the effective platform treasury and `get_pending_treasury()` returns the scheduled treasury update that has not yet taken effect, so clients no longer need to read raw storage.
>>>>>>> pr/562
=======
### Added
- CPU and memory benchmark baselines for enrollment, batch enrollment, archiving with refunds, and large-catalog registration in `docs/BENCHMARKS.md`.

### Changed
- All persistent writes now go through a shared `persist()` helper that sets the value and renews its TTL together.
- `has_completed()` now reads the enrollment's `completed` flag directly; the unreachable expiry branch and duplicated doc comment were removed.

### Removed
- Stale root `TODO.md` checklist.
>>>>>>> pr/563
=======
### Changed

- `revoke_certificate` now panics if `reason` is empty (`revocation reason cannot be empty`) or longer than 64 bytes (`revocation reason exceeds maximum length`). Callers passing an empty or oversize reason must shorten or supply one.
>>>>>>> pr/568

### Breaking change

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

- `set_arbitration_fee_config` now takes a `token` argument, and `get_arbitration_fee_config` is keyed by token. Arbitration fees are stored under `DataKey2::ArbitrationFeeConfig(token)`.
- Contract failures now return stable `Error` codes instead of string panic messages. Clients should match on `HostError: Error(Contract, #N)` and the code table in `README.md`.

### Added
- Per-token arbitration fee reserve via `DataKey2::ArbitrationFeesHeld(token)` and `get_arbitration_fees_held()`.
- Escalation credits the reserve; resolution debits it before payout.
- `withdraw_tokens()` now excludes reserved arbitration fees from the withdrawable contract balance.
- Documentation for the actual revenue split, co-instructor limitation, security model, error codes, and current test suite.

### Changed
- Enrollment counter updates now use shared `inc_total_active` / `dec_total_active` helpers.
