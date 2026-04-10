# qa-ranger Domain Pattern Library

**Version:** v1.1 — 2026-04-10

This file is a structured knowledge library loaded by the qa-ranger Strategist before any reasoning begins. It is not a skill prompt. Read every domain entry and internalize the patterns before proceeding to domain matching. When a domain match is made, cite the exact domain name from this library in the risk-strategy output so coverage decisions are traceable to a specific pattern version.

---

## Domain: Authentication / Authorization

**Examples:** Login systems, OAuth2/OIDC providers, JWT-based APIs, RBAC middleware, session management, permission gates, SSO integrations

**Critical failure modes:**
- Authorization bypass — user accesses resource they should not (IDOR, missing permission check, broken role enforcement). More common and more dangerous than authentication failures.
- Token not invalidated on logout or password change — attacker retains valid session after compromise is detected.
- Privilege escalation via mass assignment — role or admin field settable by the user through unvalidated request body.
- Broken access on soft-deleted or suspended accounts — deactivated user's token still works until expiry.
- Concurrent login race condition — two simultaneous sessions create inconsistent state on server-side session stores.

**State transitions to test:**
`unverified → verified → active → suspended → deactivated → deleted`
Token lifecycle: `issued → active → refreshed → expired → revoked`
Critical transitions: active→suspended (token must stop working immediately), password change (all existing tokens must be invalidated), role change (cached role in JWT vs. DB — may not reflect until token expiry).

**Non-obvious edge cases:**
- JWT `alg: none` — some libraries accept unsigned tokens if algorithm is not explicitly validated server-side.
- Role stored in JWT not re-checked against DB — user demoted in DB but retains elevated role until token expires.
- Password reset token not single-use — can be replayed after initial use.
- Account reactivation after email address has been reassigned to a new user.
- Admin self-demotion — removing your own admin role should be blocked (or at minimum warned) to prevent lockout.
- Session fixation — session ID not rotated on login.

**Security surface:**
- IDOR on user profile and account endpoints (accessing other users' data by ID manipulation)
- JWT algorithm confusion (`alg: none`, RS256→HS256 downgrade)
- Mass assignment on registration or profile update endpoints (role, is_admin, permissions fields)
- SSRF via password reset redirect URL parameter
- Enumeration via login/registration response timing or distinct error messages
- Concurrent session token invalidation gaps

**Performance concerns:**
- Token validation middleware runs on every authenticated request — latency impact under load
- Permission check queries on every request if roles are stored in DB rather than JWT (N+1 risk)
- Session store contention under concurrent login spikes

**UI surface:**
- Login form error message distinguishes "wrong password" from "no account" — enumeration risk via UI
- Password field autocomplete not disabled on shared-device or kiosk contexts
- "Remember me" checkbox scope not communicated — user expects session, gets permanent token
- Redirect-after-login URL parameter not validated — open redirect to attacker-controlled page
- Session timeout mid-form — user loses input silently, redirected to login with no warning
- MFA step back-navigation returns to already-completed step — state not enforced
- Password change does not inform user that other active sessions have been terminated

**Quadrant weight:**
Q1 (contract/integration): Active — token validation contracts, auth middleware integration.
Q2 (functional/scenario): Active — login flows, role-based access user journeys.
Q3 (exploratory): Active — edge cases in role enforcement, unexpected permission combinations.
Q4 (security/performance): **Dominant** — IDOR, privilege escalation, token security, auth bypass.

---

## Domain: Billing / Recurring Charges

**Examples:** SaaS subscription management, membership systems, plan upgrade/downgrade flows, payment processing (Stripe, Braintree), recurring billing cycles, invoice generation, refunds, proration

**Critical failure modes:**
- Silent double-charge on retry after network timeout — user is charged twice, no error surfaced, refund requires manual intervention and trust is damaged.
- Charge failure not surfaced — card declined but subscription remains active, revenue lost silently.
- Incorrect proration on mid-cycle plan change — proration calculation differs for upgrades vs. downgrades; downgrade path is commonly implemented incorrectly.
- Refund not applied to correct payment method — refund goes to expired card or different payment instrument than original charge.
- Subscription state desync — billing provider says active, internal DB says cancelled (or vice versa) after webhook delivery failure.
- Free trial silent conversion — trial end date passes without notification, user charged without expectation.

**State transitions to test:**
Subscription lifecycle: `trial → active → past_due → paused → cancelled → reactivated`
Payment retry cycle: `charge_attempted → failed → retry_scheduled → retry_failed → dunning → cancelled`
Critical transitions: active→cancelled (refund window, access revocation timing), past_due→active (retry success, access restoration), trial→active (first charge, no proration), plan_change mid-cycle (proration in both directions).

**Non-obvious edge cases:**
- Upgrade applied immediately but downgrade deferred to cycle end — both paths need proration tests.
- Timezone edge case on billing cycle date — user in UTC-12 billed on what they perceive as the "wrong day."
- Reactivation after cancellation at different plan tier than original — proration and pricing logic diverges.
- Concurrent subscription modification requests (upgrade + pause submitted simultaneously).
- Invoice generated before payment confirmed — invoice ID exists but charge is pending.
- Webhook replay from billing provider — idempotency key not enforced, event processed twice.

**Security surface:**
- IDOR on invoice and subscription endpoints (predictable IDs expose other users' billing data)
- Payment card details in error messages or application logs
- Subscription state manipulation via direct API call (downgrade without authorization check)
- Accessing billing history with expired or invalid token
- Webhook endpoint accepts events without signature verification

**Performance concerns:**
- Charge processing latency under load (payment gateway round-trip)
- Bulk invoice generation at cycle rollover (end-of-month spike)
- Webhook processing queue depth — delayed processing causes state desync

**UI surface:**
- Checkout submit button not disabled after first click — double-submit triggers duplicate charge attempt
- Payment card number shown unmasked after entry — PCI exposure on shared screens
- Plan upgrade/downgrade confirmation does not show proration amount — user surprised by charge
- Cancel subscription flow requires multiple obscure steps — dark pattern, also means accidental cancel is hard to catch in testing
- Session timeout mid-checkout — cart and payment details lost, no warning given before timeout
- Invoice download fails silently — PDF generation error not surfaced to user
- Price displayed without tax until final confirmation step — surprise total is a UX and trust failure

**Quadrant weight:**
Q1 (contract/integration): **Dominant** — charge idempotency, webhook processing contracts, billing provider integration.
Q2 (functional/scenario): Active — subscription management user journeys, plan change flows.
Q3 (exploratory): Active — edge cases in proration, unexpected state combinations.
Q4 (security/performance): **Dominant** — IDOR on billing data, unauthorized state manipulation, charge processing under load.

---

## Domain: User / Member Management

**Examples:** User registration and profile management, admin user CRUD, role assignment systems, team/organization membership, soft delete implementations, account merging, bulk user import

**Critical failure modes:**
- Soft delete not cascading — deleted user's data still returned in queries, relationships not cleaned up, user reappears in listings.
- Role assignment additive rather than replacement — assigning a new role does not revoke the previous one; user accumulates permissions over time.
- Relationship integrity failure after delete — orphaned records when user is deleted without cascading to dependent entities.
- Bulk import duplicate detection failure — existing users overwritten or duplicated during import.
- Account merge data loss — one account's data silently dropped during merge operation.

**State transitions to test:**
`pending_verification → active → suspended → deactivated → deleted (soft) → purged (hard)`
Critical transitions: active→suspended (access revocation must be immediate), deactivated→deleted (data retention policy triggers), deleted→reactivated (email address reuse conflicts).

**Non-obvious edge cases:**
- Reactivating a deleted account whose email has been registered by a different user.
- Admin self-demotion — removing your own admin rights; should this be blocked?
- User suspended mid-session — active session should be terminated immediately.
- Bulk import with a row that references a non-existent parent (org, team) — should fail gracefully, not silently skip.
- Soft-deleted user's email address blocked from re-registration even though account is "deleted."
- Role change not reflected in active session/token until next login.

**Security surface:**
- IDOR on user profile endpoints (accessing or modifying other users' profiles by ID)
- Mass assignment on registration (role, is_admin, verified fields settable by user input)
- Enumeration via registration endpoint (distinct error for "email already taken" vs. invalid email)
- Privilege escalation via role parameter in update endpoint
- Accessing deactivated user's data via direct API call (soft delete not enforced at API layer)

**Performance concerns:**
- Bulk user import — large file processing, duplicate detection at scale
- User search with complex filter combinations on large user tables
- Role/permission lookup on every request if not cached

**UI surface:**
- Delete account confirmation too easy to dismiss — no friction step, accidental hard delete possible
- Role assignment applied immediately with no confirmation — no undo if wrong role selected
- Bulk action (select all + delete) with no confirmation step or count shown
- Profile photo upload shows no progress — user resubmits, duplicate upload or conflicting state
- Soft-deleted user still appears in autocomplete, @mention, or assignee dropdowns
- Email change confirmation — old email confirmation link remains valid after new email confirmed
- Password change UI does not inform user that other active sessions were terminated

**Quadrant weight:**
Q1 (contract/integration): Active — role enforcement contracts, soft delete cascade integration.
Q2 (functional/scenario): **Dominant** — user lifecycle journeys, admin management flows, role assignment scenarios.
Q3 (exploratory): Active — edge cases in delete/reactivation, unexpected role combinations.
Q4 (security/performance): Active — IDOR, mass assignment, privilege escalation.

---

## Domain: Inventory / Resource Management

**Examples:** Stock management systems, availability calendars, seat reservation systems, appointment booking, resource allocation (cloud instances, licenses), product catalog with quantity tracking

**Critical failure modes:**
- Race condition on last unit — two concurrent requests both succeed for the last available item; inventory goes negative.
- Reservation expiry not firing — expired reservations not released back to available pool; inventory permanently locked.
- Bulk import overwriting live inventory — import operation resets quantity to file value, overwriting concurrent changes.
- Availability check and reservation not atomic — item shows available in check, gone by reservation time (TOCTOU).
- Negative inventory — quantity field decremented below zero with no guard.

**State transitions to test:**
`available → reserved → committed → released`
Reservation path: `pending → confirmed → expired → cancelled`
Critical transitions: reserved→expired (auto-release must fire), available→reserved (must be atomic under concurrency), committed→released (return to available pool correctly).

**Non-obvious edge cases:**
- Concurrent reservation requests for the exact last available unit — only one should succeed.
- Reservation expiry timer not firing under load — background job delayed, items not released.
- Fractional quantities (e.g., 0.5 units for time-based resources) — rounding behavior.
- Reservation for quantity greater than available — should fail with clear error, not partial fulfillment.
- Cancellation of a committed (not just reserved) resource — different refund/release logic than reservation cancel.
- Inventory count correct in DB but stale in cache — cached availability shown as available, reservation fails.

**Security surface:**
- Accessing another user's reservation details via reservation ID manipulation (IDOR)
- Unauthorized inventory adjustment via direct API call (no authorization check on quantity update)
- Bulk import endpoint accessible without admin authorization

**Performance concerns:**
- Concurrent reservation requests — database-level locking or optimistic concurrency required; latency under contention
- Bulk availability check for product catalog pages — N+1 query risk when availability is fetched per item
- Availability cache invalidation latency — stale data window under high write load

**UI surface:**
- Availability shown from stale cache — UI says available, reservation fails at confirmation step with no explanation
- Reservation countdown timer not visible or not updating — user doesn't know they have limited time
- Quantity input accepts zero or negative values via manual keyboard entry even if spinner enforces minimum
- "Only N left" indicator not updating in real-time — another user takes last unit while current user is on the page
- Concurrent booking on same item — one user gets silent failure at confirmation with no guidance on alternatives
- Cart quantity desync across browser tabs — item reserved in one tab, tab two still shows it as available

**Quadrant weight:**
Q1 (contract/integration): **Dominant** — atomic reservation contracts, concurrency handling, cache consistency.
Q2 (functional/scenario): Active — booking and reservation user journeys.
Q3 (exploratory): Active — edge cases in concurrency, expiry, and cancellation.
Q4 (security/performance): **Dominant** — concurrent access race conditions, load testing on reservation endpoints.

---

## Domain: Search / Filtering

**Examples:** Product search, user search, content search, filtered API list endpoints, faceted navigation, full-text search, paginated result sets with sort options

**Critical failure modes:**
- Stale index returning results for deleted records — soft-deleted items appear in search results.
- Sort instability on identical sort key — pagination produces duplicate or missing records when records share the same sort value.
- Inconsistent empty result handling — some endpoints return `200 []`, others return `404`, others return `200 { data: null }` for no results.
- Filter combination returning server error — untested parameter combination triggers unhandled query path.
- Cursor-based pagination breaking on concurrent inserts — new records shift page boundaries mid-traversal.

**State transitions to test:**
Less lifecycle-oriented than other domains. Key test targets are query parameter combinations and result set edge cases:
- Empty result set (no matches)
- Single result (boundary)
- Exact page boundary (last item on page N, first item on page N+1)
- Last page with fewer items than page size
- Filter that matches all records vs. filter that matches none

**Non-obvious edge cases:**
- Special characters in search query not escaped — `%`, `_`, `*`, `?` in SQL LIKE or regex patterns cause unexpected results or errors.
- Case sensitivity inconsistency — some fields case-sensitive, others not, within the same search.
- Sort direction default not matching UI label ("newest first" sorts by created_at ascending).
- Cursor-based pagination token becomes invalid when records are deleted (cursor points to deleted record).
- Filter on a null field — `status=null` vs. `status=` vs. omitting the parameter behave differently.
- Full-text search tokenization edge — hyphenated words, acronyms, leading/trailing whitespace.

**Security surface:**
- SQL injection via unparameterized filter values (especially custom sort field parameter)
- Information disclosure via filter that reveals existence of records the user should not see
- Denial of service via expensive filter combination (no query timeout or complexity limit)

**Performance concerns:**
- Unindexed filter combinations causing full table scans
- N+1 query pattern when paginated results include related data (eager vs. lazy loading)
- Deep pagination (page 1000+) with offset-based pagination — full scan to offset
- Full-text search under load — index contention during concurrent writes and reads

**UI surface:**
- Active filters not visually indicated — results are filtered but user cannot tell why
- Filter state not preserved on browser back navigation — user loses context returning from a result
- Loading state not shown on slow queries — user clicks search again, triggering duplicate request
- Search input debounce missing or misconfigured — either feels unresponsive or hammers the API
- Empty state gives no guidance — "No results" with no suggestion to adjust filters or check spelling
- Sort selection resets on page refresh or pagination — user must re-apply on every page
- Filter combination producing zero results gives no indication of which filter is the cause

**Quadrant weight:**
Q1 (contract/integration): Active — API contract for result shape, pagination token format, filter parameter schema.
Q2 (functional/scenario): **Dominant** — search and filter user journeys, result accuracy scenarios.
Q3 (exploratory): Active — unexpected filter combinations, edge cases in tokenization and sorting.
Q4 (security/performance): Active — injection via filter params, expensive query combinations.

---

## Domain: File / Media Handling

**Examples:** Document upload systems, image upload and processing, video transcoding pipelines, file storage (S3, GCS, Azure Blob), CSV import, avatar/profile photo upload, attachment handling in messaging or ticketing systems

**Critical failure modes:**
- File type validation on extension only, not MIME type — attacker uploads malicious file with `.jpg` extension.
- Size limit checked after upload completes, not before — large file accepted, stored, then rejected; storage cost incurred and DoS possible.
- Storage path constructed from user-supplied filename — path traversal attack (`../../etc/passwd`).
- Async processing failure not surfaced — virus scan or image resize fails silently, file marked available but corrupted.
- Direct URL to storage without access control — files accessible by anyone with the URL (no signed URL or auth check).

**State transitions to test:**
`uploading → uploaded → processing → available → deleted`
Failure path: `uploading → failed`, `processing → failed`
Critical transitions: uploaded→processing (async job must trigger reliably), processing→available (only after successful validation), deleted (storage deletion must occur, not just DB record removal).

**Non-obvious edge cases:**
- Zero-byte file — should it be accepted or rejected? What does the system do?
- File with no extension — MIME type detection falls back to what?
- Unicode or special characters in filename — storage key encoding, display encoding.
- Filename with path traversal characters (`../`, `..\\`, `%2F`).
- Valid extension, invalid content — `.png` file that is actually an EXE (check magic bytes, not just extension).
- Duplicate filename — overwrite or rename? Silent overwrite is a data loss risk.
- Upload interrupted mid-stream — partial file handling, cleanup of incomplete uploads.

**Security surface:**
- Path traversal via filename (`../../etc/passwd`, `%2e%2e%2f`)
- MIME type spoofing (extension vs. actual content type)
- Stored XSS via SVG upload (SVG can contain JavaScript)
- Unauthenticated access to uploaded files via direct storage URL
- Malware upload (executable files disguised as allowed types)
- Information disclosure via error messages that reveal storage path structure

**Performance concerns:**
- Large file upload blocking synchronous request processing
- Async processing queue backlog under burst upload load
- Storage retrieval latency for large files (streaming vs. buffering)
- Thumbnail generation or transcoding CPU/memory under concurrent upload load

**UI surface:**
- Upload progress not shown for large files — user assumes it froze and cancels or resubmits
- Drag-and-drop target zone not visually indicated or does not highlight on hover
- File type rejection error too generic ("Invalid file") — user not told what types are accepted
- Multi-file upload with partial failure — some files succeed, some fail, no per-file status shown
- Preview rendered before async processing completes — user sees placeholder as final result
- File size limit not communicated before upload attempt — only surfaced after wasted upload time
- Duplicate filename handling not communicated — silent overwrite is invisible data loss to the user

**Quadrant weight:**
Q1 (contract/integration): Active — upload API contract, async processing integration, storage integration.
Q2 (functional/scenario): Active — upload and retrieval user journeys.
Q3 (exploratory): Active — edge cases in filenames, content types, partial uploads.
Q4 (security/performance): **Dominant** — path traversal, MIME spoofing, malware upload, unauthenticated access, large file handling.

---

## Domain: Notifications / Messaging

**Examples:** Email notification systems, SMS delivery, push notifications, in-app notification feeds, webhook delivery systems, event-driven messaging (Kafka, SQS), transactional email (password reset, receipts, alerts)

**Critical failure modes:**
- Duplicate delivery — webhook or notification triggered multiple times due to retry without idempotency; user receives same email 3 times.
- Silent failure — notification triggered, queued, but never delivered; no error surfaced to the user or monitoring.
- Template rendering failure when required variable is null — notification sent with blank field, or crashes and is dropped silently.
- Notification sent to deactivated or deleted user — personal data sent to unreachable or invalid destination.
- Rate limit not enforced — notification storm triggered by cascading events; user receives hundreds of notifications.

**State transitions to test:**
`queued → processing → sent → delivered → read` (where applicable)
Failure path: `queued → failed → retry_scheduled → retry_failed → dead_lettered`
Critical transitions: queued→failed (error must be surfaced and logged), retry_failed→dead_lettered (must not retry indefinitely), triggered by soft-deleted resource (notification should not fire or should be suppressed).

**Non-obvious edge cases:**
- Notification triggered by soft-deleted resource — the triggering entity no longer exists when notification is processed.
- Template variable null vs. missing vs. empty string — each may behave differently in the template engine.
- User unsubscribed from notification type — unsubscribe must be respected even if event fires.
- Rate limiting per user per hour — rapid repeated actions should not flood the user.
- Notification sent during user account deactivation window — race between deactivation and notification dispatch.
- Webhook retry storm — delivery failure causes exponential retry, overwhelming the receiving endpoint.

**Security surface:**
- Notification content includes sensitive data (tokens, personal information) that should not be in email body
- Webhook endpoint accepts delivery confirmations without authentication
- Template injection — user-controlled input rendered in notification template (name field containing template syntax)
- Accessing another user's notification history via ID manipulation (IDOR on notification feed)
- Unsubscribe link that does not require authentication — anyone with the link can unsubscribe the user

**Performance concerns:**
- Notification queue depth under high event volume — processing latency increases, delayed delivery
- Bulk notification send (e.g., announcement to all users) — queue saturation, rate limiting by provider
- Template rendering at scale — complex templates with DB lookups per notification
- Dead-letter queue growth — unmonitored DLQ accumulates undelivered notifications silently

**UI surface:**
- Notification badge count not updating in real-time — stale count shown after new notifications arrive
- Notification dismissed on one device persists as unread on another — cross-device sync failure visible to user
- Unsubscribe confirmation not shown — user unsure if the action took effect
- Notification links to deleted resource — user lands on 404 with no explanation of what happened
- Notification center infinite scroll broken by new arrivals — older notifications become unreachable mid-scroll
- Mark all as read with no undo — marks notifications user has not yet opened

**Quadrant weight:**
Q1 (contract/integration): **Dominant** — queue processing contracts, delivery confirmation integration, template rendering integration.
Q2 (functional/scenario): Active — notification trigger user journeys, unsubscribe flows.
Q3 (exploratory): Active — edge cases in template rendering, retry behavior, deactivated user scenarios.
Q4 (security/performance): Active — template injection, sensitive data in notifications, queue performance under load.

---

## Adding New Patterns

When an assignment reveals a domain type not covered above, add a new entry using this template:

```markdown
## Domain: [Name]

**Examples:** [real systems or contexts this applies to]

**Critical failure modes:**
[named failures with business cost — not generic risks]

**State transitions to test:**
[primary object lifecycle with critical transitions called out]

**Non-obvious edge cases:**
[concrete, specific, surprising — the cases a junior engineer would miss]

**Security surface:**
[specific attack classes and data exposure risks for this domain]

**Performance concerns:**
[where load actually matters and why]

**UI surface:**
[failure modes and UX risks that only appear when this domain has a UI — omit if domain is API-only]

**Quadrant weight:**
[one sentence per quadrant: Active/Dominant/Inactive and why]
```

Increment the version number in the header and add the date of change. Cite the assignment that revealed the new domain in a comment on the new entry so the library's growth is traceable.
