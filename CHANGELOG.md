# Changelog

## Unreleased

### Added
- `expire_enrollment(student, course_id)` permissionlessly cleans up expired, never-completed enrollments, releases both active-enrollment counters exactly once, archives the enrollment history, and emits `enrollment_expired`.
- `refund_issued` is now emitted once for each student refunded through `process_refund()` or `archive_course()`, including the platform/instructor split.

### Breaking change
- `Enrollment.evidence_hash` and `mark_completed(..., evidence_hash)` now use `Option<BytesN<32>>` instead of `Option<String>`. Evidence is stored as a fixed 32-byte hash to reduce on-chain storage cost and keep verification deterministic.
