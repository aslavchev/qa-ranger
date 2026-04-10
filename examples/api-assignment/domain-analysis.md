# Domain Analysis

## Task Classification

**API only** — Full REST API specification provided with endpoints, HTTP methods, request/response schemas, and role-based access rules. No UI components, user journey descriptions, or interface flows detected.

---

## Business Domain

**What the system does:** FitTrack Pro is a gym membership management platform that handles member registration, recurring subscription billing, membership lifecycle (freeze, plan changes, cancellation), role-based staff and admin access, and physical gym check-ins across multiple locations.

**Who uses it:** Three actor types — gym members (self-service access to their own data and check-in history), gym staff (check-in recording, member lookup), and admins (full member and membership management including billing operations).

**What failure costs:** Billing errors on a base of 8,000 migrated members represent direct financial exposure (chargebacks, refunds, potential regulatory risk for incorrect charges). Silent failures — charges that fire without member visibility, or cancelled memberships that continue billing — are the highest-cost scenarios because they are not immediately detectable. Authorization failures exposing one member's data to another member constitute a privacy breach. Check-in failures at the gate directly block members from entering the facility and create immediate, visible service failure.

---

## Domain Pattern Match

**Domain type(s):** Billing / Recurring Charges, Authentication / Authorization, User / Member Management

**Known characteristics:**
- **Billing:** Membership status drives billing cycle. The `past_due` status indicates a failed payment — this state and its transitions to `cancelled` or recovery to `active` are not documented in the spec (gap flagged below). Freeze periods affect billing cycle — the next_billing_date must be extended by the freeze duration; this is not explicitly documented and is a known non-obvious edge case in billing domains. Annual vs. monthly billing cycles create boundary conditions at plan change time.
- **Auth:** JWT-based, short-lived access tokens (1h), refresh token rotation on `/auth/refresh`. The spec does not document refresh token expiry or whether logout invalidates all sessions or only the supplied token. IDOR risk is present on any endpoint parameterized by member ID.
- **User management:** Soft delete is confirmed (`status: deleted`). Behavior of deleted member's active memberships is not documented — this is a gap. Role enforcement is documented at the endpoint level but not validated in the spec for every case.

**Confidence:** High — all three matched domains have known patterns with confirmed characteristics in this system.

---

## Technical Surface

### API Endpoints

| Method | Path | Purpose | Key Parameters | Response |
|--------|------|---------|---------------|----------|
| POST | /auth/login | Authenticate member/staff/admin | email, password | 200 (tokens), 401 |
| POST | /auth/refresh | Refresh access token | refresh_token | 200 (new access token), 401 |
| POST | /auth/logout | Invalidate refresh token | refresh_token | 204 |
| GET | /members | List all members | status, plan_id, page, per_page | 200, 403 |
| POST | /members | Create member | email, first_name, last_name, phone, date_of_birth | 201, 409, 422 |
| GET | /members/{id} | Get single member | path: id | 200, 403, 404 |
| PUT | /members/{id} | Update member fields | any member subset | 200, 404, 422 |
| DELETE | /members/{id} | Soft-delete member | path: id | 204, 404 |
| GET | /plans | List available plans | none | 200 (public) |
| POST | /members/{id}/memberships | Create membership / subscribe | plan_id, payment_method_id, start_date | 201, 409, 422 |
| PUT | /members/{id}/memberships/{mid} | Lifecycle action (freeze/unfreeze/change_plan/cancel) | action, conditional fields | 200, 409, 422 |
| DELETE | /members/{id}/memberships/{mid} | Hard cancel membership | path: id, mid | 204 |
| POST | /check-ins | Record gym check-in | member_id, location_id | 201, 404, 409 |
| GET | /members/{id}/check-ins | List member check-in history | from, to, location_id | 200 |

### UI Flows

None detected.

### Data Models

**Member** — `id` (uuid), `email` (unique, string), `first_name`, `last_name`, `phone` (nullable), `date_of_birth` (date), `status` (`active` | `suspended` | `deleted`), `created_at`. State field: `status`.

**Plan** — `id`, `name`, `price_monthly` (decimal), `billing_cycle` (`monthly` | `annual`), `max_freeze_days` (integer), `features` (array).

**Membership** — `id`, `member_id`, `plan_id`, `status` (`active` | `frozen` | `cancelled` | `past_due`), `start_date`, `next_billing_date`, `freeze_start` (nullable), `freeze_end` (nullable), `payment_method_id`. State field: `status` with transitions: active → frozen, active → cancelled, active → past_due, frozen → active, frozen → cancelled, past_due → (undocumented).

**CheckIn** — `id`, `member_id`, `location_id`, `checked_in_at`.

---

## Requirements

### Explicit (from task input)

- "Complete test suite before we go live" — full coverage is the expectation, not sampling
- "Billing accuracy is critical — any silent billing errors on go-live would mean chargebacks and angry members on day one" — billing correctness is the highest-stakes concern, explicitly named
- "Need you to be able to confidently say this is fully tested" — coverage confidence is an explicit exit criterion
- "The billing and auth pieces especially — those are the ones that would hurt us most if something is wrong" — billing and auth are explicitly weighted as highest priority

### Implicit (inferred from domain)

- **Billing idempotency** — recurring billing systems must not double-charge; actions that trigger billing (subscription creation, plan changes) should be idempotent or protected against duplicate requests. Not mentioned in spec.
- **Freeze billing extension** — freezing a membership should push the `next_billing_date` forward by the freeze duration; failure means charging a member during a freeze period.
- **past_due recovery path** — the `past_due` status exists but no documented recovery path means a member could be stuck in `past_due` indefinitely.
- **Deleted member billing halt** — soft-deleting a member with an active membership should cancel or suspend billing; undocumented.
- **IDOR on member-scoped endpoints** — member A must not be able to access or modify member B's data via direct ID manipulation.
- **Refresh token scope** — logout should invalidate the session; whether it's single-device or all-device is undocumented.
- **Check-in gate enforcement** — a member with a `frozen`, `cancelled`, or `past_due` membership must not be able to check in.

---

## Ambiguities & Gaps

1. **`past_due` recovery path is undocumented.** The status exists in the Membership model but no endpoint or action documents how a `past_due` membership is resolved (manual admin action? automatic retry? promotion back to `active`?). This affects coverage of the full billing lifecycle.
2. **Freeze effect on `next_billing_date` is not specified.** The spec states freeze sets status to `frozen` but does not say whether `next_billing_date` is extended. If it is not, members are charged during freeze periods — a billing correctness risk.
3. **Soft-delete + active membership interaction is undocumented.** `DELETE /members/{id}` soft-deletes the member but the spec does not state what happens to an existing `active` membership. Does it cancel automatically? Continue billing? This is a data integrity gap.
4. **Refresh token expiry is not documented.** The access token expires in 3600 seconds but no expiry is documented for the refresh token. Refresh tokens that never expire are a security risk.
5. **Logout scope is not documented.** Does `POST /auth/logout` invalidate one device's session or all active sessions for the member?
6. **`suspended` member status has no creation path.** The Member model includes `status: "suspended"` but no endpoint documents how a member enters or exits the `suspended` state.
7. **`change_plan` billing timing is partially specified.** The spec says plan changes apply "at next billing cycle" but does not state whether there is a proration for the current cycle on upgrades or downgrades.
8. **Payment method validity is not validated at check.** The spec accepts a `payment_method_id` at subscription creation but does not document validation — what happens if the payment method is expired or invalid at the time of subscription?

---

## Explorer Notes

The API is well-structured and consistently role-gated. The largest coverage risk is the billing lifecycle — particularly the interaction between freeze periods and billing dates, and the `past_due` state whose recovery path is entirely undocumented. These gaps should be surfaced explicitly in the coverage summary regardless of how the strategist handles them. The IDOR surface is significant given the parameterized member ID pattern used across 8 of the 14 endpoints.
