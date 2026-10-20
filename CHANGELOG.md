# Changelog

## Unreleased

### Added
- CPU and memory benchmark baselines for enrollment, batch enrollment, archiving with refunds, and large-catalog registration in `docs/BENCHMARKS.md`.

### Changed
- All persistent writes now go through a shared `persist()` helper that sets the value and renews its TTL together.
- `has_completed()` now reads the enrollment's `completed` flag directly; the unreachable expiry branch and duplicated doc comment were removed.

### Removed
- Stale root `TODO.md` checklist.

### Breaking change
- `Enrollment.evidence_hash` and `mark_completed(..., evidence_hash)` now use `Option<BytesN<32>>` instead of `Option<String>`. Evidence is stored as a fixed 32-byte hash to reduce on-chain storage cost and keep verification deterministic.
