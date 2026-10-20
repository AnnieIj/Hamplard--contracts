# Hamplard — Contract Repo

> **On-chain course enrollment, payments, and certificate issuance on Stellar Soroban**

Hamplard is an online learning platform for practical vocational skills — tailoring, makeup artistry, baking, photography, hairstyling, nail technology, fashion design, and more. This Soroban smart contract handles the trustless financial and credential layer of the platform: course enrollment payments, on-chain instructor earnings, refunds, and verifiable certificates of completion.

## Short version

Hamplard Contracts is the Soroban smart contract layer for the Hamplard platform. It manages course lifecycle state, enrollment payments, pull-based instructor earnings, refunds, dispute fees, certificate issuance, and revocation in a trustless and auditable way.

The contract provides the core rules for:
- registering, approving, rejecting, pausing, and archiving courses
- enrolling students individually, in batches, or after a prior completion
- splitting enrollment payments between treasury and instructor earnings
- letting instructors withdraw accumulated earnings per token
- marking enrollments complete and issuing or revoking certificates
- handling refund requests, waitlists, prerequisites, and arbitration fees
- emitting events for off-chain indexing and backend synchronization

### Main contract concepts
- Courses move through `Pending`, `Active`, `Paused`, `Rejected`, and `Archived` states.
- Enrollment forwards the platform fee to treasury and credits the instructor share to an on-chain earnings balance.
- Instructors withdraw earnings per token with `withdraw_earnings()`.
- Certificates can be issued after completion and later revoked with metadata captured on-chain.

### Key features
- two-admin governance for high-impact operations
- course approval by admin or delegated approvers
- admin expiry, platform pause, student blocklist, and instructor freeze controls
- pull-based instructor earnings accounting
- batch enrollment, re-enrollment, prerequisites, capacity limits, and waitlists
- certificate issuance, revocation challenges, and bulk revocation
- per-token platform fees, risk-fee configuration, and reserved arbitration fees
- time-locked contract upgrades
- Soroban event emission for off-chain systems and indexers

The detailed documentation below expands on the architecture, data model, contract functions, and deployment flow.

This is **Repo 1 of 3** in the Hamplard project:

| Repo | Description |
|------|-------------|
| `hamplard-contract` ← you are here | Soroban smart contract (Rust) |
| `hamplard-backend` | NestJS REST API + content management |
| `hamplard-frontend` | Next.js student and instructor portal |

---

## Table of Contents

- [What the Contract Does](#what-the-contract-does)
- [What Lives Off-Chain](#what-lives-off-chain)
- [Architecture](#architecture)
- [Data Structures](#data-structures)
- [Contract Functions](#contract-functions)
- [Revenue Split Logic](#revenue-split-logic)
- [Certificate Verification](#certificate-verification)
- [Events](#events)
- [Error Codes](#error-codes)
- [Project Structure](#project-structure)
- [Prerequisites](#prerequisites)
- [Setup & Installation](#setup--installation)
- [Running Tests](#running-tests)
- [Building](#building)
- [Deploying to Testnet](#deploying-to-testnet)
- [Security Considerations](#security-considerations)
- [Roadmap](#roadmap)

---

## What the Contract Does

The Hamplard contract is the trust layer for the platform. Its core responsibilities are:

**1. Course Registry**
Instructors register courses on-chain with a price and fee structure. Each course starts as `Pending` — an admin must approve it before students can enroll. This gives the platform moderation control while payment rules remain enforced by the contract.

**2. Enrollment Payments**
When a student enrolls, the contract:
- transfers the full course price from the student into the contract
- forwards the platform fee to the treasury address
- credits the instructor share to the instructor's on-chain earnings balance

The instructor pulls that balance with `withdraw_earnings(instructor, token, amount, destination)`. This keeps payouts auditable and lets instructors choose when to withdraw. The contract records the enrollment so a student cannot enroll twice in the same course.

**3. On-Chain Certificates**
After a student completes a course (verified off-chain by the backend), the admin issues an on-chain certificate. Each certificate is a permanent, verifiable record tied to the student's Stellar address and the course. Anyone can verify a certificate by querying `verify_certificate(certificate_id)` — no trusted middleman needed. Certificates can be revoked by the admin if issued in error, subject to a challenge window.

---

## What Lives Off-Chain

The contract intentionally stores only what is necessary for payments and verification. Everything else lives in the backend database:

| Off-chain (backend) | On-chain (contract) |
|---|---|
| Video lessons | Course ID, price, instructor address |
| Course descriptions | Total enrollments, total earned |
| Assignments | Enrollment records |
| Student progress % | Completion status |
| Downloadable resources | Certificate with course title |
| User profiles | — |
| Instructor analytics | — |

This keeps the contract small, cheap to call, and focused on what blockchains are actually good at: verifiable ownership, trustless payments, and permanent records.

---

## Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                Hamplard Contract (Soroban)                    │
│                                                              │
│  ┌──────────────┐   ┌──────────────────┐   ┌─────────────┐  │
│  │   Course     │   │   Enrollment     │   │ Certificate │  │
│  │   Registry   │   │   + Payment      │   │   Issuance  │  │
│  │  (Pending →  │   │  Fee + earnings  │   │  (on-chain  │  │
│  │   Active)    │   │  credit          │   │   NFT-like) │  │
│  └──────────────┘   └──────────────────┘   └─────────────┘  │
└──────────────────────────────────────────────────────────────┘
         ↑                    ↑                    ↑
    Instructor           Student pays         Admin issues
    registers            at enroll           after backend
    course               (fee + credit)      verifies completion
```

### Roles

| Role | Address | Permissions |
|------|---------|-------------|
| **Admin** | Set at `init` | Approve/reject courses, mark completions, issue/revoke certificates, pause platform, block students, freeze instructors, configure fees, manage approvers |
| **Secondary Admin** | Set at `init` | Required alongside Admin for archive, treasury updates, admin transfer, emergency sweeps, admin expiry, and upgrades |
| **Approver** | Added by Admin | Approve or reject pending courses when delegated |
| **Instructor** | Set per course | Register courses, update/pause/unpause their own courses, withdraw earnings, transfer course ownership with admin co-approval |
| **Student** | Any address | Enroll, batch-enroll, re-enroll, request refunds, join waitlists, escalate arbitration |
| **Treasury** | Set at `init` | Receives platform fee share (passive recipient only) |

---

## Data Structures

The structs below are abridged for readability; `contracts/hamplard/src/lib.rs` is the source of truth.

### `Course`

```rust
pub struct Course {
    pub id: String,
    pub instructor: Address,
    pub price: i128,                       // token stroops; 0 = free course
    pub platform_fee_percent: u32,         // 0 = follow live token fee; otherwise a floor
    pub token: Address,
    pub total_enrollments: u32,
    pub active_enrollments: u32,
    pub total_earned: i128,
    pub status: CourseStatus,
    pub created_at_ledger: u32,
    pub max_capacity: Option<u32>,
    pub enrollment_start_ledger: Option<u32>,
    pub enrollment_expiry_ledgers: Option<u32>,
    pub min_completion_ledgers: u32,
    pub version: u32,
    pub last_updated_ledger: u32,
    pub expires_at_ledger: Option<u32>,
    pub content_hash: BytesN<32>,
    pub max_certificates: Option<u32>,
    pub certificates_issued: u32,
    pub prerequisite_course_ids: Vec<String>,
    pub archived_at_ledger: Option<u32>,
    pub categories: Vec<String>,
}
```

### `Enrollment`

```rust
pub struct Enrollment {
    pub student: Address,
    pub course_id: String,
    pub amount_paid: i128,
    pub enrolled_at_ledger: u32,
    pub completed: bool,
    pub certificate_issued: bool,
    pub certificate_id: Option<String>,
    pub evidence_hash: Option<BytesN<32>>,
    pub is_refunded: bool,
    pub course_version: u32,
    pub platform_amount: i128,             // fee actually deducted at enrollment
    pub instructor_amount: i128,           // earnings actually credited at enrollment
    pub enrollment_ref: String,
    pub completed_at_ledger: Option<u32>,
}
```

### `Certificate`

```rust
pub struct Certificate {
    pub id: String,
    pub student: Address,
    pub course_id: String,
    pub course_title: String,
    pub enrollment_reference: String,
    pub instructor: Address,
    pub issued_by: Address,
    pub issued_at_ledger: u32,
    pub revoked: bool,
    pub revoked_by: Option<Address>,
    pub revocation_ledger: Option<u32>,
    pub revocation_reason: Option<String>,
    pub expires_at_ledger: Option<u32>,
    pub instructor_signature: Option<BytesN<64>>,
    pub revocation_deadline: Option<u32>,  // Some(seq) while revocation is challengeable
}
```

### `CourseStatus` state machine

```
Pending ── approve_course() ──→ Active
Pending ── reject_course()  ──→ Rejected
Active  ── pause_course()   ──→ Paused
Paused  ── unpause_course() ──→ Active
Pending / Paused / Rejected ── archive_course() ──→ Archived
```

An `Active` course must be paused before it can be archived. `Archived` is permanent and course IDs enter a cooldown before re-registration.

---

## Contract Functions

The full public ABI is defined in `contracts/hamplard/src/lib.rs`. The main entry points are grouped below.

### `init`

```rust
pub fn init(
    env: Env,
    admin: Address,
    secondary_admin: Address,
    treasury: Address,
    default_fee_pct: u32,
    max_courses_per_instructor: u32,
    refund_window_ledgers: u32,
    revocation_challenge_period: u32,
)
```

Initialises the two-admin configuration, treasury, default fee, course limit, refund window, and certificate-revocation challenge period. Can be called once.

### Course management

| Function | Caller | Description |
|---|---|---|
| `register_course(instructor, course_id, price, token, platform_fee_pct, max_capacity, content_hash, categories)` | Instructor | Register a course in `Pending` status |
| `approve_course(caller, course_id)` | Admin or approver | Move `Pending` → `Active` |
| `reject_course(caller, course_id, reason)` | Admin or approver | Move `Pending` → `Rejected` |
| `pause_course(caller, course_id)` | Admin or instructor | Move `Active` → `Paused` |
| `unpause_course(caller, course_id)` | Admin or instructor | Move `Paused` → `Active` |
| `archive_course(admin1, admin2, course_id, students_to_refund)` | Both admins | Permanently archive; optionally refund listed students |
| `transfer_course(instructor, admin, course_id, new_instructor)` | Instructor + admin | Transfer course ownership |
| `update_course(caller, course_id, new_price, new_max_capacity)` | Admin or instructor | Update price and/or capacity |
| `update_content_hash(caller, course_id, content_hash)` | Admin or instructor | Commit a new off-chain content hash |
| `set_enrollment_start_ledger`, `set_enrollment_expiry`, `set_course_expiry` | Admin or instructor | Configure enrollment timing |
| `set_prerequisite_courses(caller, course_id, prerequisite_course_ids)` | Admin or instructor | Require completed, certified prerequisites |
| `set_max_certificates(caller, course_id, max_certs)` | Admin or instructor | Cap certificate issuance |

### Enrollment, earnings, refunds, and disputes

| Function | Caller | Description |
|---|---|---|
| `enroll(student, course_id)` | Student | Pay and enroll in one active course |
| `batch_enroll(student, course_ids)` | Student | Atomically enroll in multiple courses; returns total paid |
| `re_enroll(student, course_id)` | Student | Re-enroll after a prior completion |
| `withdraw_earnings(instructor, token, amount, destination)` | Instructor | Withdraw earnings; `amount = 0` withdraws the full balance |
| `request_refund(student, course_id)` | Student | Request a refund inside the configured window |
| `process_refund(admin, student, course_id, approved, refund_amount)` | Admin | Approve or reject a pending refund |
| `escalate_to_arbitration(caller, course_id)` | Student/caller | Pay the per-token arbitration fee and open a case |
| `resolve_arbitration(admin, caller, course_id, for_student)` | Admin | Release the held fee to the student or instructor |
| `join_waitlist`, `leave_waitlist` | Student | Manage a course waitlist |

### Completion and certificates

| Function | Caller | Description |
|---|---|---|
| `mark_completed(admin, student, course_id, evidence_hash)` | Admin | Record completion after verification |
| `issue_certificate(admin, student, course_id, certificate_id, course_title, enrollment_reference, expires_at_ledger, instructor_signature)` | Admin | Issue an on-chain certificate |
| `revoke_certificate(admin, certificate_id, reason)` | Admin | Start a revocation; it becomes permanent after the challenge period |
| `challenge_revocation(student, certificate_id)` | Certificate holder | Dispute a pending revocation during the challenge period |
| `bulk_revoke_course_certificates(admin, course_id)` | Admin | Revoke every certificate for a course |

### Governance, safety, and upgrades

| Function | Caller | Description |
|---|---|---|
| `transfer_admin(admin1, admin2, new_admin, new_secondary_admin)` | Both admins | Propose a new admin pair |
| `accept_admin(new_admin, new_secondary_admin)` | New admin pair | Accept the transfer and clear admin expiry |
| `update_treasury(admin1, admin2, new_treasury)` | Both admins | Schedule a treasury update after a delay |
| `set_admin_expiry(admin1, admin2, expires_at)` | Both admins | Set or clear the admin expiry ledger |
| `withdraw_tokens(admin1, admin2, token, amount, destination)` | Both admins | Emergency sweep; reserved arbitration fees are excluded |
| `pause_platform` / `unpause_platform` | Admin | Halt or restore all enrollments |
| `block_student` / `unblock_student` | Admin | Global student blocklist |
| `freeze_instructor` / `unfreeze_instructor` | Admin | Block an instructor's course and earnings actions |
| `add_approver` / `remove_approver` | Admin | Delegate course approval authority |
| `propose_upgrade` / `cancel_upgrade` / `upgrade_contract` | Both admins | Time-locked Wasm upgrade lifecycle |
| `set_upgrade_timelock(admin1, admin2, ledgers)` | Both admins | Configure the upgrade review window |

### Fee and risk configuration

| Function | Caller | Description |
|---|---|---|
| `set_fee_config(admin, token, fee_bps)` | Admin | Set per-token platform fee in basis points |
| `get_fee_config(token)` | Anyone | Read per-token fee, falling back to the default |
| `update_default_fee(admin, new_fee_pct)` | Admin | Update the global fallback fee percentage |
| `set_arbitration_fee_config(admin, token, fee_per_case)` | Admin | Set the arbitration fee for a token |
| `get_arbitration_fee_config(token)` | Anyone | Read the per-token arbitration fee |
| `get_arbitration_fees_held(token)` | Anyone | Read fees reserved for open arbitration cases |
| `set_risk_config_enabled` / `set_risk_fee_config` | Admin | Configure optional risk surcharges |
| `get_effective_fee_for_payment(student, token, payment_amount)` | Anyone | Preview the effective fee breakdown |
| `add_approved_token` / `remove_approved_token` | Admin | Manage accepted payment tokens |

### Read-only queries

| Function | Returns |
|---|---|
| `get_course(course_id)` | `Option<Course>` |
| `get_enrollment(caller, student, course_id)` | `Option<Enrollment>` |
| `get_enrollment_history(caller, student, course_id)` | Enrollment history visible to the caller |
| `get_certificate(caller, certificate_id)` | `Certificate` (student, instructor, or admin only) |
| `get_course_certificates(course_id)` | Certificate IDs for a course |
| `is_enrolled(student, course_id)` | `bool` |
| `has_completed(student, course_id)` | `Option<bool>` |
| `verify_certificate(certificate_id)` | `bool` — exists, not revoked, not expired |
| `list_courses(offset, limit)` | Paginated course IDs |
| `get_instructor_earnings(instructor, token)` | Withdrawable earnings for one token |
| `get_instructor_total_earnings(instructor)` | Total earnings across tokens |
| `get_instructor_stats(instructor)` | Enrollment/completion/certificate counts |
| `get_courses_by_instructor(instructor)` | Course IDs owned by an instructor |
| `get_refund_request(student, course_id)` | `Option<RefundRequest>` |
| `get_arbitration_case(caller, course_id)` | `Option<ArbitrationCase>` |
| `get_waitlist(course_id)` | `Option<Vec<Address>>` |

---

## Revenue Split Logic

Enrollment is a single atomic contract invocation. The full course price moves from the student to the contract, then the contract:

1. forwards the platform fee to treasury, and
2. credits the instructor's share to `DataKey::InstructorEarnings(instructor, token)`.

The instructor share is **not pushed to the instructor at enrollment**. It stays in the contract's accounting until the instructor calls `withdraw_earnings()`.

### Fee calculation

The platform fee uses basis points (bps), where `10000 bps = 100%`:

```
base_fee_bps      = FeeConfig(token).fee_bps, or DefaultFee * 100 as fallback
risk_surcharge_bps = optional configured surcharge
effective_fee_bps = min(base_fee_bps + risk_surcharge_bps, 10000)

platform_amount   = price * effective_fee_bps / 10000
instructor_amount = price - platform_amount
```

The course-level `platform_fee_pct` argument is a **percent** from `0` to `100`:

- `0` means “follow the live per-token fee”.
- A non-zero value must be at least the current token fee and acts as a **floor** on the live fee at enrollment.

The current risk resolver returns `(false, false)` for both risk flags, so `enroll()` and `re_enroll()` do not add a surcharge in this version even when risk configuration is enabled. The preview and enrollment paths share the same calculation helper.

### Example: 100 USDC course at a 20% fee

USDC uses 7 decimal places, so `100 USDC = 1_000_000_000 stroops`.

```
price             = 1_000_000_000 stroops  (100 USDC)
effective_fee_bps = 2000                    (20%)

platform_amount   = 1_000_000_000 * 2000 / 10000
                  = 200_000_000 stroops     (20 USDC) → treasury

instructor_amount = 1_000_000_000 - 200_000_000
                  = 800_000_000 stroops     (80 USDC) → instructor earnings
```

The student pays 100 USDC. Treasury receives 20 USDC immediately, and the instructor can withdraw 80 USDC with:

```bash
stellar contract invoke \
  --id <CONTRACT_ID> \
  --source instructor-account \
  --network testnet \
  -- withdraw_earnings \
  --instructor <INSTRUCTOR_ADDRESS> \
  --token <USDC_SAC_ADDRESS> \
  --amount 0
```

`amount = 0` withdraws the full available balance. The optional `destination` argument lets the instructor send earnings to another address.

### Refund accounting

Each enrollment stores the `platform_amount` and `instructor_amount` actually used at payment time. Refunds split funds in those stored proportions, so a later fee-config change cannot alter an existing enrollment's refund calculation.

### Co-instructors

`CoInstructor { address, share_bps }` is defined in the contract types, but `Course` does not store co-instructors and no entry point splits or credits their shares. Every enrollment is currently split only between treasury and the single `Course.instructor`. Co-instructor revenue shares are not supported yet.

---

## Certificate Verification

Any third party — an employer, a client, a government agency — can verify a Hamplard certificate without trusting the platform:

```bash
stellar contract invoke \
  --id <CONTRACT_ID> \
  --network testnet \
  -- verify_certificate \
  --certificate_id "CERT-12345-TAILORING"
```

Returns `true` if the certificate exists, has not expired, and has not been permanently revoked. A certificate with a pending revocation remains verifiable until its challenge deadline passes. The `get_certificate()` function returns the full record including the student's Stellar address, course title, instructor, and issuance ledger — all permanently recorded and independently auditable.

---

## Events

The contract emits granular events for indexers. The most important ones are:

| Event name | Payload | When |
|---|---|---|
| `initialized` | `(admin, secondary_admin, treasury, default_fee_pct, max_courses_per_instructor, refund_window_ledgers)` | Contract initialised |
| `course_registered` | `course_id` | Instructor registers a course |
| `course_approved` | `(course_id, instructor, caller, ledger)` | Admin or approver approves a course |
| `course_rejected` | `(course_id, instructor, caller, reason, ledger)` | Admin or approver rejects a course |
| `course_paused` / `course_unpaused` | `(course_id, status, caller, ledger)` | Course status changes |
| `course_archived` | `(course_id, admin1, admin2)` | Course permanently archived |
| `course_updated` | `(course_id, version, caller)` | Price/capacity changed |
| `course_transferred` | `(course_id, previous_instructor, new_instructor, admin, ledger)` | Course ownership transferred |
| `student_enrolled` | `(student, course_id, price, platform_amount, instructor_amount, ledger)` | Student enrolls |
| `batch_enrolled` | `(student, course_ids, total_paid)` | Atomic batch enrollment |
| `platform_fee_transferred` | `(treasury, amount, ledger)` | Platform fee forwarded |
| `instructor_payment_transferred` | `(instructor, amount, ledger)` | Instructor earnings credited |
| `earnings_withdrawn` | `(token, amount, recipient)` | Instructor withdraws earnings |
| `course_completed` | `(student, admin, ledger)` | Completion recorded |
| `certificate_issued` | `(student, course_id, admin, issued_at_ledger)` | Certificate issued |
| `certificate_revoked` | `(admin, certificate_id, student, course_id, reason, ledger, deadline)` | Revocation started |
| `course_certificates_revoked` | `(admin, course_id, revoked_count, ledger)` | Bulk revocation |
| `refund_requested` / `refund_processed` | Refund details | Refund lifecycle |
| `dispute_escalated` / `arbitration_resolved` | Arbitration details | Dispute lifecycle |
| `tokens_withdrawn` | `(admin1, admin2, token, amount, destination)` | Emergency sweep |
| `upgrade_proposed` / `upgrade_executed` / `upgrade_cancelled` | Upgrade details | Timelocked upgrade lifecycle |

See `contracts/hamplard/src/lib.rs` for the complete event list, including waitlist, freeze, blocklist, approver, and configuration events.

---

## Error Codes

Every contract failure is reported as a stable numeric `Error` code. Soroban surfaces these as:

```
HostError: Error(Contract, #N)
```

Clients should match on `N`; the numeric values are part of the public interface and do not change once assigned.

| Code | Name | Meaning |
|---:|---|---|
| 1 | `AlreadyInitialized` | `init` has already been called |
| 2 | `Unauthorized` | Caller is missing the required role or authorization |
| 3 | `AdminExpired` | The admin role has reached its configured expiry |
| 4 | `InvalidAdminConfig` | Invalid admin pair, proposal, or transfer state |
| 5 | `InvalidFeeConfig` | Fee rate, bps value, or arbitration fee is invalid |
| 6 | `InvalidTreasury` | Treasury address is invalid or not configured |
| 7 | `CourseNotFound` | No active or archived course exists for the ID |
| 8 | `InstructorFrozen` | The instructor is frozen |
| 9 | `InstructorState` | Invalid freeze/unfreeze state transition |
| 10 | `ArchivedCourseImmutable` | An archived course cannot be modified |
| 11 | `InvalidCourseInput` | Malformed course ID, capacity, categories, or hash |
| 12 | `InvalidPrice` | Price is outside the allowed precision range |
| 13 | `CourseAlreadyRegistered` | Course ID is already in use |
| 14 | `CourseIdInCooldown` | Archived course ID is still in its re-registration cooldown |
| 15 | `InstructorCourseLimitReached` | Instructor has reached the course limit |
| 16 | `GovernanceWindowNotElapsed` | Required approval/review delay has not elapsed |
| 17 | `CourseNotPending` | Operation requires a pending course |
| 18 | `HighValueApprovalConflict` | The same approver cannot provide both high-value approvals |
| 19 | `CounterUnderflow` | A counter would be decremented below zero |
| 20 | `CourseNotActive` | Operation requires an active course |
| 21 | `CourseNotPaused` | Operation requires a paused course |
| 22 | `InvalidCourseTransfer` | Invalid course ownership transfer |
| 23 | `CourseMustBePausedToArchive` | Active courses must be paused before archiving |
| 24 | `TooManyRefundStudents` | Refund list exceeds the maximum size |
| 25 | `ArithmeticOverflow` | Checked arithmetic overflow |
| 26 | `InvalidPrerequisite` | Prerequisite is invalid, missing, or incomplete |
| 27 | `EmptyBatch` | Batch enrollment list is empty |
| 28 | `BatchTooLarge` | Batch or list exceeds the maximum size |
| 29 | `DuplicateCourseInBatch` | Batch contains duplicate course IDs |
| 30 | `PlatformPaused` | Platform enrollment is paused or pause state is invalid |
| 31 | `StudentBlocked` | Student is blocked or block state is invalid |
| 32 | `EntryRestricted` | Address is not allowed to enroll or join a waitlist |
| 33 | `InstructorReferenceInvalid` | Invalid instructor reference |
| 34 | `CourseNotAvailable` | Course is not enrollable (status, timing, capacity, token, or cap) |
| 35 | `AlreadyEnrolled` | Student is already enrolled |
| 36 | `CircuitBreakerTripped` | Repeated payment failures auto-paused the course |
| 37 | `TransferAmountMismatch` | Token transfer delivered an unexpected amount |
| 38 | `EnrollmentNotFound` | No enrollment exists for the student/course pair |
| 39 | `EnrollmentStateError` | Completion or enrollment-state transition is invalid |
| 40 | `InvalidWithdrawal` | Withdrawal amount, destination, or balance is invalid |
| 41 | `InvalidCertificateInput` | Malformed certificate field |
| 42 | `CertificateAlreadyIssued` | Certificate already exists or was already issued |
| 43 | `MaxCertificatesReached` | Course certificate cap reached |
| 44 | `CertificateError` | Revocation, challenge, or certificate lookup failed |
| 45 | `RefundError` | Refund window, request state, or amount is invalid |
| 46 | `ApproverState` | Approver add/remove operation conflicts with current state |
| 47 | `ArbitrationError` | Arbitration config or case state is invalid |
| 48 | `WaitlistError` | Waitlist operation is invalid |
| 49 | `UpgradeError` | Upgrade proposal, timelock, or execution state is invalid |
| 50 | `InconsistentState` | Internal invariant or reserved-balance check failed |

---

## Project Structure

```
hamplard-contract/
├── Cargo.toml                         ← Rust workspace config
├── Cargo.lock
├── .gitignore
├── README.md
└── contracts/
    └── hamplard/
        ├── Cargo.toml                 ← Contract package config
        ├── Makefile                   ← Build / deploy shortcuts
        └── src/
            ├── lib.rs                 ← Full contract logic
            └── test.rs                ← 313 unit tests
```

---

## Prerequisites

```bash
# Rust + wasm32 target
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
rustup target add wasm32v1-none

# Stellar CLI
cargo install --locked stellar-cli --features opt

# Testnet account
stellar keys generate --global my-account --network testnet
stellar keys fund my-account --network testnet
```

---

## Setup & Installation

```bash
git clone https://github.com/your-org/hamplard-contract.git
cd hamplard-contract
cargo check
```

---

## Running Tests

```bash
cargo test
cargo test -- --nocapture          # with output
cargo test test_full_lifecycle     # single test
```

Expected summary:

```
test result: ok. 313 passed; 0 failed; 0 ignored
```

---

## Building

The Makefile lives in `contracts/hamplard`:

```bash
cd contracts/hamplard
make build      # → target/wasm32v1-none/release/hamplard.wasm
make optimize   # → target/wasm32v1-none/release/hamplard.optimized.wasm
```

---

## Deploying to Testnet

```bash
cd contracts/hamplard
export STELLAR_ACCOUNT=my-account
make deploy-testnet
# → CXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
```

### Initialize after deployment

Run the following from `contracts/hamplard`:

```bash
stellar contract invoke \
  --id <CONTRACT_ID> \
  --source my-account \
  --network testnet \
  -- init \
  --admin <ADMIN_ADDRESS> \
  --secondary_admin <SECONDARY_ADMIN_ADDRESS> \
  --treasury <TREASURY_ADDRESS> \
  --default_fee_pct 20 \
  --max_courses_per_instructor 25 \
  --refund_window_ledgers 17280 \
  --revocation_challenge_period 1440
```

### Register and approve a test course

```bash
# Register
stellar contract invoke \
  --id <CONTRACT_ID> \
  --source instructor-account \
  --network testnet \
  -- register_course \
  --instructor <INSTRUCTOR_ADDRESS> \
  --course_id "COURSE-TAILORING-001" \
  --price 500000000 \
  --token <USDC_SAC_ADDRESS> \
  --platform_fee_pct 0 \
  --max_capacity 100 \
  --content_hash <64_HEX_CHARACTERS> \
  --categories '["tailoring"]'

# Approve
stellar contract invoke \
  --id <CONTRACT_ID> \
  --source my-account \
  --network testnet \
  -- approve_course \
  --caller <ADMIN_OR_APPROVER_ADDRESS> \
  --course_id "COURSE-TAILORING-001"
```

> **USDC SAC on Testnet:** `CBIELTK6YBZJU5UP2WWQEUCYKLPU6AUNZ2BQ4WWFEIE3USCIHMXQDAMA`

---

## Security Considerations

### Authorization and governance

- Every state-changing entry point calls `require_auth()` on the relevant party.
- Two admin keys are stored separately. High-impact operations — archiving, treasury updates, admin transfer, emergency sweeps, admin expiry, and upgrades — require both admins.
- `add_approver()` lets the admin delegate course approval and rejection without delegating treasury or certificate authority.
- `set_admin_expiry()` can block all admin operations at a ledger height. Recovery requires the existing admins to nominate a new pair via `transfer_admin()`, followed by `accept_admin()`.
- Admin transfers have a cooldown, treasury changes take effect after a 100-ledger delay, and upgrades have a configurable two-admin time-lock (default 17,280 ledgers).

### Enrollment and funds

- Duplicate enrollment is rejected, and `batch_enroll()` is all-or-nothing.
- Enrollment validates course status, enrollment window, capacity, prerequisite certificates, token whitelist, platform pause, student blocklist, and platform-wide caps before any payment.
- Payment is atomic: the full price is transferred, the platform fee is forwarded to treasury, and the instructor share is credited to the instructor's earnings balance. A failed transfer leaves no enrollment state behind.
- Instructor earnings are pull-based and tracked per token. Only the instructor must authorize `withdraw_earnings()`; the contract address cannot be used as the destination.
- Refunds use the `platform_amount` and `instructor_amount` stored at enrollment time, so later fee changes do not alter existing refunds.
- Arbitration fees are tracked in a per-token reserve. Escalation collects the exact configured fee, resolution releases it, and `withdraw_tokens()` cannot sweep reserved fees.
- A per-course circuit breaker pauses a course after repeated failed payment attempts.
- `block_student`, `freeze_instructor`, and `pause_platform` provide emergency controls without deleting historical records.

### Validation and auditing

- Fee bps values are capped at 10,000 (100%). Course prices are either zero or within 0.01–100,000 USDC at 7-decimal precision.
- Course IDs, titles, categories, capacity, content hashes, and certificate fields are length- and format-validated.
- Certificate revocation is challengeable for a configured period. Revocation metadata (admin, ledger, reason) is retained permanently, and revoked certificates are never deleted.
- `verify_certificate()` returns true for a certificate with a pending revocation until the challenge deadline passes, false after a permanent revocation, and false after certificate expiry.
- All sensitive state changes emit events for off-chain monitoring.

### Known limitations

- **Off-chain content:** the contract cannot enforce that course materials are delivered or that their quality matches expectations.
- **Instructor insolvency:** if instructor earnings are insufficient for a refund path, that path can fail. Callers should verify balances before bulk refunds.
- **Token risk:** balances are denominated in the enrolled token; the contract makes no exchange-rate guarantee.
- **Admin compromise:** both admin keys are required for the highest-impact operations, but a compromised single admin can still perform single-admin actions until rotation or expiry.
- **Risk surcharges:** risk-fee configuration exists, but the current risk resolver returns no flags, so enrollment uses the base fee only.
- **Co-instructors:** co-instructor shares are defined as a type but are not stored or paid out yet.

---

## Roadmap

- [x] Course registry with approval and rejection workflow
- [x] Enrollment payment with platform fee + instructor earnings credit
- [x] Instructor earnings withdrawal tracking
- [x] Batch enrollment and re-enrollment
- [x] On-chain certificate issuance, revocation, and challenge window
- [x] Course pause/unpause and permanent archive
- [x] Refund requests with admin approval
- [x] Waitlists, prerequisites, capacity limits, and enrollment windows
- [x] Arbitration fee configuration and per-token fee reserves
- [x] Two-admin governance, approvers, admin expiry, blocklist, and instructor freeze
- [x] Time-locked contract upgrade mechanism
- [x] 313 unit tests
- [ ] Discount codes / coupon system (Soroban-based)
- [ ] Subscription enrollment model (monthly access)
- [ ] Co-instructor revenue shares and payouts
- [ ] Mainnet deployment

---

## License

MIT
