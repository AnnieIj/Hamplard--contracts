# Changelog

## Unreleased

### Storage layout change

- `Course` gains two fields, `total_refunded: i128` and `refund_count: u32`, updated by both refund paths (`process_refund` and `archive_course`) so admins can read per-course refund rates on-chain via `get_course`. `Course` is a `#[contracttype]` struct, so courses stored before this change lack the new keys and will not deserialize against the new type; redeploy fresh, or migrate existing `Course` / `ArchivedCourse` entries before upgrading a live contract.

### Breaking change

- `Enrollment.evidence_hash` and `mark_completed(..., evidence_hash)` now use `Option<BytesN<32>>` instead of `Option<String>`. Evidence is stored as a fixed 32-byte hash to reduce on-chain storage cost and keep verification deterministic.
