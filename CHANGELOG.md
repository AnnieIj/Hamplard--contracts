# Changelog

## Unreleased

<<<<<<< HEAD
<<<<<<< HEAD
### Storage layout change

- `Course` gains two fields, `total_refunded: i128` and `refund_count: u32`, updated by both refund paths (`process_refund` and `archive_course`) so admins can read per-course refund rates on-chain via `get_course`. `Course` is a `#[contracttype]` struct, so courses stored before this change lack the new keys and will not deserialize against the new type; redeploy fresh, or migrate existing `Course` / `ArchivedCourse` entries before upgrading a live contract.
=======
### Changed

- `revoke_certificate` now panics if `reason` is empty (`revocation reason cannot be empty`) or longer than 64 bytes (`revocation reason exceeds maximum length`). Callers passing an empty or oversize reason must shorten or supply one.
>>>>>>> pr/568
=======
### Added
- Refunds now release a course capacity seat by decrementing `total_enrollments`; `max_capacity` remains a lifetime cap on non-refunded enrollments.
- `update_min_completion_ledgers()` now rejects values above `MAX_MIN_COMPLETION_LEDGERS` (1,000,000 ledgers).
- `expire_enrollment(student, course_id)` permissionlessly cleans up expired, never-completed enrollments, releases both active-enrollment counters exactly once, archives the enrollment history, and emits `enrollment_expired`.
- `refund_issued` is now emitted once for each student refunded through `process_refund()` or `archive_course()`, including the platform/instructor split.
>>>>>>> pr/573

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


### Fixed
- Define `InstructorTotalEarnings` as lifetime net earnings: refunds reduce it, withdrawals do not, and the getter documents the distinction from the withdrawable per-token balance. (#344)
- Frozen instructors can no longer withdraw earnings until an admin unfreezes them. (#342)
- Correct the enrollment-expiry documentation: expiry does not remove the record or free a capacity slot. (#339)
- Allow `re_enroll()` to replace an expired enrollment, archiving the expired record to `EnrollmentHistory`. (#338)


### Tests
- Add targeted `mock_auths` coverage for privileged and fund-moving entrypoints, asserting exactly which signers each operation requires instead of relying on `mock_all_auths()`.
