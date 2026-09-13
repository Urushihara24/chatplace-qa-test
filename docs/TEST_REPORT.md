# TEST REPORT / RELEASE READINESS REPORT

**Feature:** Keyword Chatbot (Telegram) · ChatPlace.io  
**Candidate:** Vsevolod Samoylov · **Date:** 30.07.2026 · **Report version:** 1.1  
**AI used during preparation:** Qwen for structure and wording; scope, prioritization, Risk Matrix, and Go/No-Go decisions were my own.  
**Full registry (AC ↔ cases ↔ defects ↔ risks):** `test-artifacts/` in this repository.

## 0. Reasoning approach
Testing was performed from a release-gatekeeper perspective rather than as a simple happy-path walkthrough. The order was: define responsibility boundaries and the API↔UI contract → run risk-based checks around paid onboarding and payment, the most expensive funnel points → validate runtime behavior for “keyword → content delivery”. Shift-left was applied by reviewing the contract before UI execution. RCA used a 5 Whys loop to localize the failure point in the service chain, with hypotheses tested iteratively through direct actions. Traceability from AC ↔ cases ↔ defects ↔ evidence was maintained by ID, similar to a TMS workflow such as Allure TestOps or Zephyr.

> Execution of steps 2–3 on the environment was **blocked** by `BUG-01`. The relevant cases were designed from product logic and correctly marked *Blocked*, not *Not done*: test design does not depend on environment availability.

## 1. Scope and responsibility boundaries
- **Within our control (ChatPlace):** paywall, token validation, onboarding, UI feedback, storing the binding in the database, sync-service health and alerting, handling external-provider responses, and the HTTP status returned to the client.
- **Outside our control (Telegram Bot API):** whether Telegram reports the bot as a channel administrator, message delivery, and platform rate limits such as 30 msg/s and 4096 characters.
- **Outside our control (external payment provider):** provider-side decline or timeout is an expected external event. **Within our control:** how our backend maps that provider response and which status is returned from `api.chatplace.io`; a 500 for an expected decline is a defect in our adapter/contract, not in the provider.
- **Outside our control (external analytics):** Yandex Metrica WebSocket errors (`wss://mc.yandex.com`) are outside the product boundary and do not block release; they are filtered as noise.
- **Outside our control (user actions):** permissions granted to the bot, subscription/block state, and card balance. The product can still provide clear guidance in the UI.

## 2. Acceptance Criteria
- **AC-1 — Subscription-gated access and subscription creation.** Without a plan the user sees a paywall; with a plan the step is accessible; a payment decline is handled as 4xx without 5xx.
- **AC-2 — Token-based binding.** A valid token produces a “Connected” state within ≤ 3 s; an invalid or revoked token produces a clear error and does not create the bot.
- **AC-3 — Bot-admin synchronization.** `GET /bots/{id}/telegram-channels` returns the channel when the bot is confirmed as an administrator.
- **AC-4 — Onboarding can be completed.** Transition 1→2→3 works without hanging; empty/error responses produce an empty/error state with a “Retry” action.
- **AC-5 — Keyword behavior.** The keyword is saved, validated for empty values, duplicates, and case, and triggers in the defined context such as direct messages or comments.
- **AC-6 — Content delivery.** A subscriber receives the content within ≤ 5 s after the trigger; text/file/link formats render correctly.
- **AC-7 — Exception handling.** There is no silent failure in the UI: every unsuccessful state provides feedback and a next action.
- **AC-8 — Technical hygiene on paid screens.** The console contains no SDK errors/warnings and payment options are applied according to the Definition of Done.

## 3. Checklist — condensed
The full registry contains 18 cases in `test-artifacts/`.

- **Connection (AC-1, AC-2):** paid access opens only with a subscription · a valid token binds the bot · invalid/empty/revoked tokens are rejected with clear text · there is a visible way to replace or remove the token.
- **Onboarding and synchronization (AC-3, AC-4, AC-7):** after “Bot added”, `telegram-channels` returns a non-empty array when the bot is actually an administrator · when `data:[]`, the UI shows an empty state with guidance and retry instead of remaining silent · “Bot added” does not allow progression without an actual binding · transition 1→2→3 can be completed normally.
- **Runtime (AC-5, AC-6):** keyword triggers on exact match · boundaries include case, spaces, special characters, Unicode/emoji · repeated trigger behavior matches the specification · content is delivered within ≤ 5 s · text above 4096 characters is blocked in the UI before submission.
- **Negative / integration (AC-1, AC-3, AC-7):** payment decline such as insufficient funds or invalid CVC returns backend 4xx with a typed code, without 500 or misleading UI · token revoked after binding produces a graceful error and no 5xx · Telegram API unavailability emulated through Charles Proxy produces retry behavior without data loss · rate limiting during mass triggers is queued rather than crashing the service.
- **Technical hygiene (AC-8):** Network response for `telegram-channels` matches the contract; payment-page Console does not contain `Unsupported PaymentOptions` or deprecated-attribute warnings.

## 4. Testing approach
Shift-left: requirements and API contract reviewed before UI checks. API: Postman collection for binding/synchronization endpoints plus direct Telegram Bot API calls such as `getChatMember` to independently verify administrator status. E2E: real Telegram account, trigger, then content-delivery verification. Negative testing: timeouts/429 emulated with Charles Proxy; insufficient-funds card used as a normal payment-decline scenario. Defect localization: Kibana logs plus sync-service state, with RCA through 5 Whys and iterative hypothesis validation to isolate the failing layer in the service chain. Required defect evidence package: steps + Expected/Actual + logs/screenshots. Security: secrets such as JWT `x-auth-token` and Stripe `pk_live` were removed from evidence before publication in line with the security checklist. Regression: checklist execution on staging before release.

## 5. Tools
Postman · Swagger/OpenAPI · DevTools (Network/Console) · Charles Proxy · Kibana · Telegram + @BotFather · Jira · Allure TestOps · GitLab CI.

## 6. Defects found
Detailed RCA is available in the registry.

- **BUG-01 [Critical].** Onboarding cannot be completed: `200 OK` + `{type:"result", data:[]}` is returned for a valid bot administrator; the UI blocks the step without feedback. RCA through 5 Whys: the “dead sync service” hypothesis was **disproved by action** — the container was running and the bot was reachable, while the defect still reproduced. The failure point therefore shifted toward the API↔UI contract / missing database binding: the backend was not reflecting the real Telegram administrator state.
- **BUG-02 [Medium].** No visible mechanism exists to replace/remove the token after binding, leaving a user stuck after entering an incorrect token.
- **BUG-03 [High].** Payment decline: the business layer handled the decline correctly — response body `{type:"error", data:{code:0, message:…, level:"business"}}` identified the business decline and produced a message — but HTTP mapping returned `500` instead of 4xx, while `code:0` did not provide a useful typed error to the client. This was not “the server is down” and not an unhandled exception; the defect was narrower: `level → HTTP status` mapping plus missing error-code specification. The 500 came from our `api.chatplace.io` domain, not from the provider. The flow itself was not completely broken because the frontend displayed the body message, but the contract and DoD were violated and error typing was lost. The misleading “try again later” copy was a consequence of `code:0`. RCA was refined through three iterations as more response details became available.
- **OBS-01 [Trivial].** Payment SDK warnings such as `saveCard/debug/sbpSupport` plus deprecated `allowpaymentrequest` appeared in the payment console, so some payment options may not be applied. This **violates the Definition of Done** requirement for a clean console. A text dump is stored in `evidence/`.

## 7. Release decision — NO-GO
The release is **blocked** because `BUG-01` prevents users from activating a purchased feature through the normal paid-onboarding path. This creates direct churn and support risk at the most expensive point in the funnel.

**Conditions for Go:** `BUG-01` is fixed and verified, the bound channel is returned correctly, onboarding completes end to end, no Critical/High defects remain open, and paid screens satisfy the DoD with a clean console, correctly configured payment SDK, and payment declines returning 4xx.

External conditions outside our control, such as Telegram API availability, provider-side decline, and third-party analytics, do not block release by themselves but should be monitored.

**Severity is calibrated by UX outcome rather than domain:** the frontend successfully falls back to a readable decline message for payment, so `BUG-03 = High`; onboarding remains silent and blocks progress on an empty response, so `BUG-01 = Critical`. The NO-GO decision is primarily caused by `BUG-01`; `BUG-03` strengthens the decision because it is a second defect in the paid flow and violates the DoD, but it is not the sole release blocker.