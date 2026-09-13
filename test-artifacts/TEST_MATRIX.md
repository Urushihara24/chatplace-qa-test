# TEST MATRIX · Keyword Chatbot (Telegram)

Registry of test artifacts. ID traceability: `AC-xx` ↔ `TC-xx` ↔ `BUG-xx`/`OBS-xx` ↔ `evidence/`.  
Test case statuses: **Pass** / **Fail** / **Blocked** / **Not run**.  
Candidate: Vsevolod Samoylov · 30.07.2026.

---

## 1. AC_Traceability — acceptance criteria ↔ checks

| ID | Acceptance Criteria | Acceptance condition | Cases |
|---|---|---|---|
| AC-1 | Subscription-gated access + subscription creation | Without a plan → paywall; with a plan → step is available; payment decline is handled as 4xx without 5xx | TC-01, TC-02, TC-18 |
| AC-2 | Token-based binding | Valid token → “Connected” within ≤3 s; invalid/revoked token → error | TC-03, TC-04, TC-05 |
| AC-3 | “Bot = admin” synchronization | `GET /telegram-channels` returns the channel when administrator status is real | TC-06, TC-07 |
| AC-4 | Onboarding is completable | Transition 1→2→3; empty response → empty/error state + retry | TC-08, TC-09 |
| AC-5 | Keyword | Validated for empty/duplicate/case and triggers in the specified context | TC-10, TC-11, TC-12 |
| AC-6 | Content delivery | Subscriber receives content within ≤5 s and the format renders correctly | TC-13, TC-14 |
| AC-7 | Exception handling | No silent UI failure: every unsuccessful state provides feedback + next step | TC-09, TC-15, TC-16 |
| AC-8 | Technical hygiene on paid screens | No SDK errors/warnings in Console; payment options are applied according to DoD | TC-17 |

---

## 2. Test_Cases — execution checklist

| ID | Group | Check | P | AC | Status | Link |
|---|---|---|---|---|---|---|
| TC-01 | Paywall | Without a subscription, the configuration screen is protected by the paywall | P1 | AC-1 | Pass | — |
| TC-02 | Paywall | With a subscription, step 1/3 is available | P1 | AC-1 | Pass | — |
| TC-03 | Token | Valid token binds the bot and shows “Connected” | P1 | AC-2 | Pass | — |
| TC-04 | Token | Invalid/empty token is rejected with clear text | P1 | AC-2 | Not run | blocked by BUG-01 |
| TC-05 | Token | A visible path exists to replace/delete the token | P2 | AC-2 | Fail | BUG-02 |
| TC-06 | Sync | Bot is a channel administrator → `GET /telegram-channels` is non-empty | P1 | AC-3 | Fail | BUG-01 |
| TC-07 | Sync | Bot is NOT an administrator → clear message rather than an unexplained empty array | P2 | AC-3 | Blocked | BUG-01 |
| TC-08 | Onboarding | Transition 1→2→3 works normally | P1 | AC-4 | Blocked | BUG-01 |
| TC-09 | Onboarding | When `data:[]`, UI shows an empty state + retry instead of remaining silent | P1 | AC-4,7 | Fail | BUG-01 |
| TC-10 | Trigger | Keyword triggers on exact match | P1 | AC-5 | Blocked | — |
| TC-11 | Trigger | Boundaries: case, spaces, special characters, emoji | P2 | AC-5 | Blocked | — |
| TC-12 | Trigger | Empty/duplicate keyword is not saved | P2 | AC-5 | Blocked | — |
| TC-13 | Delivery | Content is delivered within ≤5 s after the trigger | P1 | AC-6 | Blocked | — |
| TC-14 | Delivery | Text/file/link renders correctly in Telegram | P2 | AC-6 | Blocked | — |
| TC-15 | Negative | Token revoked after binding → graceful error, no 5xx | P2 | AC-7 | Blocked | — |
| TC-16 | Negative | Telegram API unavailable through Charles → retry without data loss | P2 | AC-7 | Blocked | — |
| TC-17 | Hygiene | Payment Console contains no Unsupported PaymentOptions / deprecated warnings | P3 | AC-8 | Fail | OBS-01 |
| TC-18 | Payment negative | Card decline → backend 4xx with typed code, not 500; UI message matches the error type without misleading copy | P1 | AC-1 | Fail | BUG-03 |

> Cases marked **Blocked** were designed from product logic; execution was blocked by `BUG-01` at step 1/3. This is not “unfinished work” but a documented dependency.

---

## 3. Defect_Register — summary

| ID | Title | Sev | Prio | Type | Area | Status | Evidence |
|---|---|---|---|---|---|---|---|
| BUG-01 | Onboarding cannot be completed with a valid bot administrator | Critical | High | Functional + integration + UX | Paid onboarding, step 1/3 | Open | bug01_*.png |
| BUG-02 | No visible token replace/delete mechanism | Medium | Medium | UX | Bot connection | Open | — |
| BUG-03 | Payment decline: business layer handled `level:"business"`, but HTTP mapping returned 500 instead of 4xx; `code:0` does not type the error | High | High | Server 5xx + integration + UX | Payment flow / AC-1 | Open | bug03_*.png |

### BUG-01 — detailed
- **Steps:** 1) Purchase a subscription and enter the token. 2) Add the bot to the channel as an administrator; permissions are confirmed in Telegram. 3) Click “Bot added”.
- **Expected:** `GET /telegram-channels` returns the channel; or an empty response produces an empty/error state + retry; transition to 2/3 is possible.
- **Actual:** API returns `200 OK` with `{type:"result", data:[]}` despite confirmed administrator status; UI gives no feedback and blocks progress.
- **RCA (5 Whys):** the “dead sync service” hypothesis was **disproved by action** — the container was running and the bot was reachable while the defect still reproduced. Failure point moved to API↔UI contract / missing database binding, with the backend not reflecting the real Telegram administrator state.
- **Impact:** a paying user cannot activate the purchased feature → churn and support-ticket risk at the most expensive funnel point.

### BUG-02 — detailed
- **Steps:** 1) Bind the bot using a token. 2) Look for a UI action to replace or remove the token.
- **Expected:** a clear “Replace/Delete token” operation is available without creating a new account.
- **Actual:** no such element was found; the recovery path after entering a wrong token is unclear.
- **Impact:** the user gets stuck and is likely to open a support ticket.

### BUG-03 — detailed
- **Precondition:** a card without sufficient funds causes CloudPayments to decline the payment. This was reproduced as a negative scenario; no real successful payment was required.
- **Steps:** 1) Open subscription checkout through CloudPayments. 2) Use card data that produces insufficient-funds decline. 3) Confirm payment.
- **Expected:** decline is returned as 4xx with a specified business code such as `insufficient_funds`, `code != 0`, and `level:"business"`; frontend displays the response `message` plus an appropriate action such as another card/top-up; Console contains no 5xx.
- **Actual:** HTTP `500 Internal Server Error` is returned from `POST /subscription/create-cloudpayments`, while the body is `{type:"error", data:{code:0, message:"Insufficient funds on the card", level:"business"}}`. The business layer **did execute** — it identified the decline, generated a message, and classified it as `level:"business"`; this was NOT an unhandled exception. Violations: (1) HTTP mapping wraps a business error as 5xx instead of 4xx; (2) `code:0` does not specify an error type, so the frontend cannot distinguish decline categories programmatically.
- **RCA iterations:** “dead service” → disproved; “user is completely stuck” → disproved by UI check because the frontend renders the response message; “unhandled exception” → disproved by the body because `level:"business"` shows business logic executed. Final localization: defect in `level → HTTP status` mapping plus missing `code` specification.
- **Responsibility boundary:** 500 comes from `api.chatplace.io`, our backend HTTP-mapping layer, not CloudPayments. Decline itself is external; mapping the response level to an HTTP status is within product responsibility. Provider server-to-server response is not visible from the browser and was not evaluated.
- **Open question, not an accusation:** `level:"business"` makes intentional 500 unlikely. If it is intentional, it should be documented as an architecture decision and DoD/alerting should be revised; either way, the current status conflicts with expected HTTP semantics and the project Definition of Done.
- **UX copy issue:** the body `message` is correct, but the subtitle “Please try again later” conflicts with it. This is frontend hardcoded fallback caused by `code:0` preventing typed guidance.
- **Additional checks:** compare a funded-card happy path and expired-card/invalid-CVC negatives by `code`/`level`/HTTP status; verify that a broken subscription is not created after the 500 response.
- **Correlation with OBS-01:** the CloudPayments endpoint is a likely source of `Unsupported PaymentOptions (sbpSupport)` warnings; causality was not proven.
- **Impact:** 5xx in a monetization flow violates DoD and creates alert fatigue around expected declines, hiding real incidents; `code:0` breaks client-side error typing. Severity is **High, not Critical** because the business logic and user path remain partially functional through frontend fallback. `BUG-01`, not `BUG-03`, is the main NO-GO blocker.

---

## 4. Observations — technical debt / non-critical findings

| ID | Title | Sev | Type | Symptom | Risk | Recommendation | Status |
|---|---|---|---|---|---|---|---|
| OBS-01 | Payment SDK warnings during checkout | Trivial | Config / technical debt | Console: Unsupported PaymentOptions (saveCard/debug/sbpSupport); deprecated allowpaymentrequest | Some payment options such as SBP or saved card may not be applied | Update SDK / verify Payment Element configuration; satisfy the “clean console” DoD | Open |
| OBS-02 | “Bot added” does not validate that the bot was actually added | Medium | UX / validation | Button can be clicked without really adding the bot to the channel | False success state → dead end | Validate administrator status server-side before unlocking the next step | Open |

---

## 5. Risk_Matrix — probability × impact

| AC | Area | Failure probability | Impact | Risk level | Test priority | Rationale |
|---|---|---|---|---|---|---|
| AC-1 | Paywall / payment | **High** | **High** | **High** | **P1** | BUG-03: decline → backend 500 from `api.chatplace.io`, masked by frontend fallback; DoD violation and misleading-copy risk. Path is not fully broken, so not Critical; NO-GO is driven by BUG-01 |
| AC-2 | Token binding | Medium | High | High | P1 | External input + integration is a common failure source |
| AC-3 | “Bot = admin” synchronization | **High** | **High** | **Critical** | **P0** | Depends on database binding and Telegram API; BUG-01 was found here |
| AC-4 | Onboarding completion | **High** | **High** | **Critical** | **P0** | Paying user gets stuck without feedback; direct churn risk |
| AC-5 | Keyword | Medium | High | High | P1 | Core feature with many input boundaries |
| AC-6 | Content delivery | Medium | High | High | P1 | Core product value; delivery integration |
| AC-7 | Exception handling | High | Medium | High | P1 | Silent UI failure loses users; confirmed by BUG-01 |
| AC-8 | Technical hygiene / DoD | Medium | Low | Low | P3 | Technical debt; does not block the feature but violates DoD |

> Read the matrix this way: P0/Critical areas are tested first and can block release; see the Go/No-Go section in `docs/TEST_REPORT.md`. The blocking defect `BUG-01` is concentrated exactly in the P0 areas AC-3 and AC-4. AC-1 was raised to P1/High after `BUG-03`, but not to P0, because payment fallback preserves the user path while onboarding remains silent on the empty response. Severity is calibrated by real UX outcome rather than by domain, which confirms the risk-based prioritization.