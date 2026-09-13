# EVIDENCE — Defect Evidence Base

Screenshots and text dumps supporting defects and observations from `test-artifacts/TEST_MATRIX.md`. The filenames below are the actual attached image names and the same names referenced from the registry, preserving traceability by ID.

> **Security note for a public repository.** All DevTools screenshots were sanitized before publication: JWT `x-auth-token: eyJ0…` was kept outside the frame, Stripe `pk_live_…` is not visible, and `project_id` / `bot_id` can be redacted where needed. Only evidence-relevant, non-secret information remains: HTTP status, response body, channel administrator list, and warning/error text. This follows the security checklist for secrets and data in transit: a public repository must not contain tokens, including my own test tokens.

## Evidence index

| File | Linked to | What it proves | Reproduction path | Redacted |
|---|---|---|---|---|
| `bug01_empty_channels.png` | BUG-01, TC-06, AC-3 | DevTools → Network → `GET /telegram-channels`: General/Headers shows the request path and `Status Code 200 OK` | F12 → Network → click “Bot added” → `telegram-channels` request → Headers | `X-Auth-Token` kept outside the frame |
| `bug01_empty_response.png` | BUG-01, TC-06, TC-09, AC-3/AC-4 | Response/Preview for the same request: `{type:"result", data:[]}` — an empty array despite confirmed administrator status | same request → Response tab | not required; no secrets present |
| `bug01_bot_is_admin.png` | BUG-01 contrast evidence | Telegram → channel → Administrators: `TestCalc — admin`. Proves the empty array is not caused by the user failing to grant administrator rights | Telegram → channel → title → Administrators | not required |
| `bug03_cloudpayments_500.png` | BUG-03, TC-18, AC-1 | DevTools → Network/Headers: `POST …/create-cloudpayments` returns red `500 Internal Server Error`, which does not match the business-level decline outcome | F12 → Network → confirm payment with insufficient funds → `create-cloudpayments` request → Headers | `X-Auth-Token` kept outside the frame |
| `bug03_response_body.png` | BUG-03, TC-18, AC-1 | Response for the same request: `{type:"error", data:{code:0, message:"Insufficient funds on the card", level:"business"}}`. Shows that the business layer handled the decline while HTTP mapping was wrong; `code:0` means the client receives no useful typed error | same request → Response tab | not required; no secrets present |
| `bug03_decline_ui.png` | BUG-03 contrast evidence, TC-18, AC-1 | UI for the same decline: fallback with “Insufficient funds on the card” plus “Retry”. Shows that the frontend rendered the response message while the subtitle remained hardcoded and contradictory | same scenario → screen after payment decline | not required |

## How to read the contracts
- **BUG-01 — two sides of the contract:** `bug01_bot_is_admin.png` proves that the bot is actually a channel administrator on the external platform. `bug01_empty_channels.png` + `bug01_empty_response.png` show that ChatPlace still returns `200 OK` with an empty array and blocks onboarding without feedback. The mismatch “platform = yes / product = no” is the core defect. The machine-readable counterpart is represented by requests in `api/chatplace_keyword_bot.postman_collection.json`.
- **BUG-03 — broken `level → HTTP status` contract:** `bug03_cloudpayments_500.png` shows HTTP 500, `bug03_response_body.png` shows `level:"business", code:0`, and `bug03_decline_ui.png` shows the frontend rendering the body message with a hardcoded subtitle. Together they prove that business logic executed and the frontend read the message, but the HTTP status does not match the response level and `code:0` prevents useful client-side error typing. This is not a dead end or a crashed server; it is a focused mapping defect.

## OBS-01 — Console dump
Text evidence captured on the payment/subscription screen from payment SDK chunks `CNGlEfIp.js` / `paymentblocks.js`:

```text
CNGlEfIp.js:28 Unsupported 'PaymentOptions' field: 'saveCard'
CNGlEfIp.js:28 Unsupported 'PaymentOptions' field: 'debug'
CNGlEfIp.js:28 Unsupported 'PaymentOptions' field: 'sbpSupport'
VM821 paymentblocks.js:1 Allow attribute will take precedence over 'allowpaymentrequest'.
```

Interpretation: the product passes fields that the current payment-widget SDK does not recognize, which suggests an outdated SDK or incorrect configuration; `allowpaymentrequest` is deprecated. This violates the Definition of Done requirement for no console errors/unhandled UI issues. Risk: some payment options such as SBP or saved-card behavior may not be applied. No secret appears in the dump; `pk_live` is not present in these lines, so a binary screenshot is not required.

## Artifact status
BUG-01 and BUG-03 screenshots are stored in this folder and indexed above. OBS-01 is represented by the text dump, which is sufficient for a Trivial observation and safer than a screenshot that could accidentally expose keys. A full HAR session dump is **not committed** because of secret-sanitization requirements; a sanitized copy can be provided separately when required.