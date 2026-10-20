# Changelog

## Unreleased

### Added
- Refunds now release a course capacity seat by decrementing `total_enrollments`; `max_capacity` remains a lifetime cap on non-refunded enrollments.
- `update_min_completion_ledgers()` now rejects values above `MAX_MIN_COMPLETION_LEDGERS` (1,000,000 ledgers).
- `expire_enrollment(student, course_id)` permissionlessly cleans up expired, never-completed enrollments, releases both active-enrollment counters exactly once, archives the enrollment history, and emits `enrollment_expired`.
- `refund_issued` is now emitted once for each student refunded through `process_refund()` or `archive_course()`, including the platform/instructor split.

### Breaking change
- `Enrollment.evidence_hash` and `mark_completed(..., evidence_hash)` now use `Option<BytesN<32>>` instead of `Option<String>`. Evidence is stored as a fixed 32-byte hash to reduce on-chain storage cost and keep verification deterministic.
