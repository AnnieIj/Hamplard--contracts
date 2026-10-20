# Changelog

## Unreleased

### Breaking changes
- `Enrollment.evidence_hash` and `mark_completed(..., evidence_hash)` now use `Option<BytesN<32>>` instead of `Option<String>`. Evidence is stored as a fixed 32-byte hash to reduce on-chain storage cost and keep verification deterministic.
- `freeze_instructor(admin, instructor, reason)` and `block_student(admin, student, reason)` now require a non-empty reason of at most 256 bytes. Their events include the reason.
- `get_instructor_total_earnings(instructor)` is removed because it mixed balances denominated in different tokens. Use `get_instructor_lifetime_earnings(instructor, token)` instead.

### Added
- `get_enrollment_status(student, course_id)` returns `None`, `Active`, `Expired`, `Completed`, or `Refunded`, including refunds recovered from enrollment history.
- `get_instructor_freeze_reason(instructor)` and `get_student_block_reason(student)` expose stored moderation reasons.

### Fixed
- Free-course enrollments no longer consume the platform-wide paid-enrollment cap or decrement its counter on completion/refund.
- Instructor lifetime net earnings are tracked per token; withdrawals do not change them and refunds debit only the refunded token.

### Migration
- Existing `InstructorTotalEarnings` entries are no longer read. Historic cross-token totals cannot be decomposed into per-token amounts, so per-token lifetime counters start from zero after upgrade; refunds use saturating subtraction so pre-upgrade enrollments remain refundable.
