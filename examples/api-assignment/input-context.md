# Input Context — FitTrack Pro API Assignment

*This file represents the raw inputs passed to qa-ranger. In a real run, these would be provided directly at invocation.*

---

## Free Text Input (Email from Product Manager)

> Hi,
>
> We're going live with FitTrack Pro across 12 gym locations next month and I need a complete test suite for the membership management API before we cut over.
>
> The API handles everything: member registration, plan subscriptions, billing cycles, membership freezes, and gym check-ins. We're migrating roughly 8,000 active members from our old system so billing accuracy is critical — any silent billing errors on go-live would mean chargebacks and angry members on day one.
>
> The system will have staff and admin roles in addition to regular members. Staff can check members in and view member details. Only admins can create or delete members and change plans.
>
> I need you to be able to confidently say this is fully tested before we sign off. The billing and auth pieces especially — those are the ones that would hurt us most if something is wrong.
>
> Let me know if you need the API spec, I can send it over.
>
> Thanks,
> Jordan

---

## URL Input

`https://docs.fittrackpro.internal/api/v1/reference`

*In a real run, the Explorer would fetch this URL. The content below represents what was returned:*

---

## Fetched: FitTrack Pro API v1 Reference

**Base URL:** `https://api.fittrackpro.internal/v1`
**Auth:** Bearer token (JWT). Include `Authorization: Bearer <token>` on all protected endpoints.

---

### Authentication

**POST /auth/login**
- Body: `{ "email": string, "password": string }`
- Response 200: `{ "access_token": string, "refresh_token": string, "expires_in": 3600 }`
- Response 401: `{ "error": "invalid_credentials" }`

**POST /auth/refresh**
- Body: `{ "refresh_token": string }`
- Response 200: `{ "access_token": string, "expires_in": 3600 }`
- Response 401: `{ "error": "invalid_or_expired_token" }`

**POST /auth/logout**
- Body: `{ "refresh_token": string }`
- Response 204: no body
- Invalidates the supplied refresh token.

---

### Members

**GET /members** *(admin only)*
- Query params: `status`, `plan_id`, `page`, `per_page` (default 20, max 100)
- Response 200: `{ "data": [Member], "meta": { "total": int, "page": int } }`
- Response 403: non-admin caller

**POST /members** *(admin only)*
- Body: `{ "email": string, "first_name": string, "last_name": string, "phone": string (optional), "date_of_birth": date }`
- Response 201: Member object
- Response 409: email already exists
- Response 422: validation errors

**GET /members/{id}** *(admin or staff, or the member themselves)*
- Response 200: Member object
- Response 403: caller is a different member
- Response 404: member not found

**PUT /members/{id}** *(admin only)*
- Body: any subset of member fields
- Response 200: updated Member object
- Response 404: member not found
- Response 422: validation errors

**DELETE /members/{id}** *(admin only)*
- Soft delete — sets status to `deleted`, does not remove record
- Response 204: no body
- Response 404: member not found

---

### Plans

**GET /plans** *(public — no auth required)*
- Response 200: `{ "data": [Plan] }`

---

### Memberships

**POST /members/{id}/memberships** *(admin only)*
- Body: `{ "plan_id": string, "payment_method_id": string, "start_date": date (optional, defaults to today) }`
- Response 201: Membership object
- Response 409: member already has an active or frozen membership
- Response 422: validation errors

**PUT /members/{id}/memberships/{mid}** *(admin only)*
- Actions (via `action` field in body):
  - `freeze`: `{ "action": "freeze", "freeze_end": date }` — sets status to `frozen`; requires active membership; `freeze_end` must be ≤ `max_freeze_days` from today
  - `unfreeze`: `{ "action": "unfreeze" }` — sets status back to `active`; requires frozen membership
  - `change_plan`: `{ "action": "change_plan", "plan_id": string }` — changes plan at next billing cycle; requires active membership
  - `cancel`: `{ "action": "cancel" }` — sets status to `cancelled`; requires active or frozen membership
- Response 200: updated Membership object
- Response 409: invalid state transition
- Response 422: validation errors

**DELETE /members/{id}/memberships/{mid}** *(admin only)*
- Hard cancel — immediately sets status to `cancelled` and stops billing
- Response 204: no body

---

### Check-ins

**POST /check-ins** *(staff or admin)*
- Body: `{ "member_id": string, "location_id": string }`
- Response 201: CheckIn object
- Response 404: member not found
- Response 409: member has no active membership (cannot check in)

**GET /members/{id}/check-ins** *(admin, staff, or the member themselves)*
- Query params: `from`, `to` (ISO date), `location_id`
- Response 200: `{ "data": [CheckIn] }`

---

### Data Models

**Member**
```
id: uuid
email: string (unique)
first_name: string
last_name: string
phone: string | null
date_of_birth: date
status: "active" | "suspended" | "deleted"
created_at: datetime
```

**Plan**
```
id: uuid
name: string
price_monthly: decimal
billing_cycle: "monthly" | "annual"
max_freeze_days: integer
features: string[]
```

**Membership**
```
id: uuid
member_id: uuid
plan_id: uuid
status: "active" | "frozen" | "cancelled" | "past_due"
start_date: date
next_billing_date: date
freeze_start: date | null
freeze_end: date | null
payment_method_id: uuid
```

**CheckIn**
```
id: uuid
member_id: uuid
location_id: uuid
checked_in_at: datetime
```
