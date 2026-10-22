# Changelog

## Unreleased

### Breaking change
- `Enrollment.evidence_hash` and `mark_completed(..., evidence_hash)` now use `Option<BytesN<32>>` instead of `Option<String>`. Evidence is stored as a fixed 32-byte hash to reduce on-chain storage cost and keep verification deterministic.
- `enroll(student, course_id, max_price)`, `batch_enroll(student, course_ids, max_price)` and `re_enroll(student, course_id, max_price)` now take a `max_price` argument and revert when the course's current price exceeds it, so a price change between quote and execution can never charge a student more than they approved (#361).
