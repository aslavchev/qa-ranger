# Risk Strategy

## Domain Pattern Match

**Pattern library version:** v1.0 — 2026-04-08

**Matched domains:**
- Billing / Recurring Charges — Known pattern — System has subscription plans, recurring billing cycles, freeze periods affecting billing dates, a `past_due` state, and plan-change events that trigger billing recalculation. All core billing pattern characteristics confirmed.
- Authentication / Authorization — Known pattern — JWT access + refresh token pair, role-based access control (member/staff/admin), IDOR-vulnerable parameterized member ID endpoints, logout invalidation. All core auth pattern characteristics confirmed.
- User / Member Management — Known pattern — Full CRUD with soft delete, role assignment, multi-actor access model, relationship between member record and dependent membership records. All core user-management pattern characteristics confirmed.

**Unmatched areas:** None.

**Partial match gaps:** None. All three matched domains have full pattern coverage for this system's characteristics.

---

## Quadrant Coverage Map

**Q1 (Technology-facing, supports team):** Active — API contract validation (schema correctness, status code accuracy, role enforcement on every protected endpoint), state transition integrity testing for the Membership lifecycle, and idempotency verification for subscription creation.

**Q2 (Business-facing, supports team):** Inactive — No UI components detected; no user journey tests applicable.

**Q3 (Business-facing, critiques product):** Inactive — No UI components detected.

**Q4 (Technology-facing, critiques product):** Active — Authorization bypass and IDOR testing across all member-parameterized endpoints, injection surface on free-text fields (email, name), and concurrency testing for duplicate subscription creation under simultaneous requests.

---

## Risk Areas (ranked by business impact)

### Risk 1: Membership Billing State Transitions

**Business impact:** Silent billing errors — charging during a freeze, double-charging on plan change, or continuing to charge after cancellation — directly produce chargebacks from 8,000 migrated members on day one of go-live. The PM explicitly named billing errors as the primary financial and reputational risk. These failures are not immediately visible to members and require manual reconciliation to detect and reverse.

**Impact tier:** Irreversible / high-cost — financial loss, chargeback exposure, regulatory risk on incorrect charges.

**Domain pattern source:** Billing / Recurring Charges — Critical failure modes ("charge fires during freeze," "double-charge on plan change"), State transitions (active → frozen → active, active → past_due), Non-obvious edge cases ("freeze does not extend next_billing_date — member charged during freeze period").

**Techniques selected:**
- State transition testing — the Membership status field (`active`, `frozen`, `cancelled`, `past_due`) has defined legal and illegal transitions; each must be verified independently, including the undocumented `past_due` state.
- Boundary value analysis — `freeze_end` must not exceed `max_freeze_days` from today; the boundary at exactly `max_freeze_days` and at `max_freeze_days + 1` are the critical values.
- Exploratory — the freeze → billing date interaction and the plan change proration behavior are undocumented; exploratory scenarios must probe these to surface what the API actually does.

**Quadrant(s):** Q1, Q4

**Priority tier:** P0

---

### Risk 2: Authentication and Authorization Controls

**Business impact:** An authorization failure allowing member A to read or modify member B's data constitutes a privacy breach across 8,000 member records. An auth bypass allowing unauthenticated access to admin-only endpoints would expose the entire member database. The PM explicitly named auth as the second highest risk. Given the IDOR surface across 8 parameterized endpoints, this is a systematic risk, not a one-off scenario.

**Impact tier:** Irreversible / high-cost — privacy breach, potential regulatory exposure (GDPR/data protection), reputational damage.

**Domain pattern source:** Authentication / Authorization — Security surface ("IDOR on member-scoped endpoints," "token replay after logout"), Critical failure modes ("member accesses another member's record via ID manipulation").

**Techniques selected:**
- IDOR / authorization testing — every endpoint that accepts a member ID must be tested with a caller whose ID does not match the path parameter. Three role pairs to cover: member-as-other-member, staff-as-admin-endpoint, unauthenticated caller.
- State transition testing — token lifecycle: valid → expired → refreshed → logged out → replay attempt. Each transition must be tested.

**Quadrant(s):** Q1, Q4

**Priority tier:** P0

---

### Risk 3: Member Lifecycle and Soft Delete Integrity

**Business impact:** A soft-deleted member with an active membership that continues billing would be a silent billing error (see Risk 1) compounded by the fact the member no longer appears active in the system, making detection harder. Creating a member with a duplicate email after soft-delete could collide with the existing record and corrupt data integrity.

**Impact tier:** Recoverable / medium-cost — data integrity issue that can be corrected but requires investigation to detect.

**Domain pattern source:** User / Member Management — Non-obvious edge cases ("soft delete does not cancel dependent records automatically," "email uniqueness enforcement after soft delete").

**Techniques selected:**
- State transition testing — member status transitions: active → deleted; what happens to the linked membership?
- Equivalence partitioning — email field: valid unique email (happy path), duplicate active email (409), duplicate soft-deleted email (behavior undocumented — must probe).

**Quadrant(s):** Q1

**Priority tier:** P1

---

### Risk 4: Subscription Creation and Plan Validation

**Business impact:** A member successfully subscribing with an invalid or expired payment method would create a billing record that immediately enters `past_due` with no recovery path documented — effectively creating a broken subscription on day one. Allowing a member to hold two active memberships simultaneously (race condition at subscription creation) would create a billing duplication.

**Impact tier:** Recoverable / medium-cost — broken subscription state that requires admin intervention; not silent but requires manual resolution.

**Domain pattern source:** Billing / Recurring Charges — Critical failure modes ("second subscription created during concurrent requests"), Security surface ("payment method validity not verified at subscribe time").

**Techniques selected:**
- Equivalence partitioning — valid payment method (happy path), invalid payment method id, expired payment method, already-active membership (409 expected).
- Concurrency / exploratory — simultaneous POST /members/{id}/memberships requests to probe whether duplicate subscription creation is prevented at the database level.

**Quadrant(s):** Q1, Q4

**Priority tier:** P1

---

### Risk 5: Check-in Gate Enforcement

**Business impact:** A member with a frozen, cancelled, or `past_due` membership successfully checking in would bypass physical access control — a direct facility security failure. On a 12-location rollout, gate enforcement errors would be immediately visible to staff and members.

**Impact tier:** Recoverable / medium-cost — visible service failure but correctable by staff; no financial exposure.

**Domain pattern source:** Inventory / Resource Management (partial application) — access control enforcement at point of service. Also covered by the check-in 409 response for "member has no active membership."

**Techniques selected:**
- Equivalence partitioning — member status partitions for check-in: active membership (201 expected), frozen membership (409 expected), cancelled membership (409 expected), past_due membership (409 expected), no membership at all (404 or 409 expected — spec says 409 but ambiguous).
- State transition testing — membership just cancelled moments before check-in attempt.

**Quadrant(s):** Q1

**Priority tier:** P2

---

## Coverage Confidence

**Overall:** Medium

**Justification:** All three matched domains have full pattern coverage and all active risk areas have applicable techniques, but three undocumented behaviors — the `past_due` recovery path, the freeze-to-billing-date interaction, and the soft-delete-to-active-membership interaction — mean coverage in these areas relies on exploratory probing rather than specification-based testing; the outcomes of those test cases cannot be asserted in advance.

**Coverage gaps:**
1. `past_due` membership recovery — no documented API path to recover from `past_due` status; test cases can reach the state and observe behavior but cannot assert the correct outcome without clarification.
2. Freeze → `next_billing_date` extension — whether freezing extends the billing date is undocumented; test cases will probe the field value but the correct assertion is unknown without spec clarification.
3. Soft-delete + active membership interaction — behavior is undocumented; test cases will observe what happens but correct behavior cannot be pre-specified.
4. `suspended` member status — no documented creation or exit path; excluded from coverage (see Global Exclusions).

---

## Writer Brief

### Work Queue (ordered by priority)

1. Membership Billing State Transitions
2. Authentication and Authorization Controls
3. Member Lifecycle and Soft Delete Integrity
4. Subscription Creation and Plan Validation
5. Check-in Gate Enforcement

---

### Per-Area Specifications

#### Membership Billing State Transitions

**Priority tier:** P0
**Technique(s):** state-transition, boundary, exploratory
**Quadrant(s):** Q1, Q4
**Required scenarios:**
- Active membership freeze with `freeze_end` exactly at `max_freeze_days` limit (boundary — valid)
- Active membership freeze with `freeze_end` one day beyond `max_freeze_days` (boundary — rejected with 409 or 422)
- Freeze action on an already-frozen membership (invalid state transition — 409)
- Unfreeze action on an active membership (invalid state transition — 409)
- Cancel action on an active membership (valid — 200, status → cancelled)
- Cancel action on a frozen membership (valid — 200, status → cancelled)
- Cancel action on an already-cancelled membership (invalid state transition — 409)
- Attempt freeze after cancellation (invalid state transition — 409)
- Exploratory: observe `next_billing_date` before and after freeze to determine if it is extended
- Exploratory: plan change via `change_plan` action on active membership — observe whether current cycle is prorated
**Test case depth:** 6–8 test cases, majority negative (invalid transitions and boundary violations)
**Exclusions:** `past_due` recovery path — behavior is undocumented; write one exploratory scenario to observe what the endpoint returns if a `past_due` action is attempted, but do not assert a specific correct outcome. Note the gap explicitly.

---

#### Authentication and Authorization Controls

**Priority tier:** P0
**Technique(s):** idor, state-transition
**Quadrant(s):** Q1, Q4
**Required scenarios:**
- Member authenticates and accesses their own record (GET /members/{id}) — 200 expected
- Member attempts to access a different member's record via ID manipulation — 403 expected, no data leaked
- Staff attempts to access admin-only endpoint (POST /members) — 403 expected
- Unauthenticated request to any protected endpoint — 401 expected
- Expired access token used without refresh — 401 expected
- Valid refresh token used to obtain new access token — 200 with new token
- Refresh token replayed after logout — 401 expected (token invalidated)
- Member attempts to list all members (GET /members, admin-only) — 403 expected
**Test case depth:** 5–7 test cases, majority negative (unauthorized and invalid token scenarios)
**Exclusions:** Multi-session logout behavior (all-device vs. single-device) — behavior is undocumented; note the gap but do not write assertions that assume either behavior.

---

#### Member Lifecycle and Soft Delete Integrity

**Priority tier:** P1
**Technique(s):** state-transition, equivalence
**Quadrant(s):** Q1
**Required scenarios:**
- Create member with valid unique email — 201, all fields returned
- Create member with duplicate email (existing active member) — 409
- Create member with duplicate email of a soft-deleted member — probe behavior (undocumented; observe and note)
- Soft-delete a member with no active membership — 204, member status becomes `deleted`
- Soft-delete a member with an active membership — 204; observe membership status after (undocumented — note gap)
- Attempt to update a deleted member — expected 404 or 422; observe behavior
**Test case depth:** 4–6 test cases, mixed positive and negative
**Exclusions:** `suspended` status — no creation or exit path documented; excluded entirely. Note the gap.

---

#### Subscription Creation and Plan Validation

**Priority tier:** P1
**Technique(s):** equivalence, concurrency
**Quadrant(s):** Q1, Q4
**Required scenarios:**
- Subscribe a member with a valid plan and valid payment method — 201, membership status `active`
- Subscribe a member who already has an active membership — 409
- Subscribe a member who already has a frozen membership — 409
- Subscribe with a non-existent plan_id — 404 or 422 (observe behavior)
- Subscribe with a non-existent payment_method_id — observe behavior (undocumented; note gap)
- Concurrent subscription creation: two simultaneous POST requests for the same member — only one should succeed with 201, the other 409
**Test case depth:** 4–6 test cases, majority negative
**Exclusions:** Payment method validation (expired card, declined) — behavior at subscription time is undocumented; write one exploratory scenario, do not assert specific outcome.

---

#### Check-in Gate Enforcement

**Priority tier:** P2
**Technique(s):** equivalence
**Quadrant(s):** Q1
**Required scenarios:**
- Check in a member with an active membership at a valid location — 201
- Check in a member with a frozen membership — 409
- Check in a member with a cancelled membership — 409
- Check in a member with no membership — 409 (spec) or 404 (ambiguous — observe and note)
- Check in a non-existent member — 404
**Test case depth:** 3–5 test cases, majority negative
**Exclusions:** Multi-location concurrency (two simultaneous check-ins for the same member at different locations) — out of scope for this assignment; no concurrency documentation provided.

---

### Global Exclusions

- Performance and load testing — the PM did not specify load requirements or thresholds; performance testing without defined SLAs produces unactionable results. Excluded from this suite.
- `GET /plans` endpoint — public, no auth, read-only, no state effects; risk is negligible. One smoke test is sufficient and is covered implicitly by the subscription creation test setup.
- Pagination correctness on `GET /members` — functional but low business risk; excluded to keep scope on high-impact areas. The PM did not mention it.
- Check-in history (`GET /members/{id}/check-ins`) date filtering — low risk, no business-critical behavior. Excluded.
