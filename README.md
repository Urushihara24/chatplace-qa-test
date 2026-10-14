# ChatPlace Keyword Chatbot — QA Release Assessment

Release-readiness QA assessment of a commercial Telegram automation product: paid onboarding, bot/channel binding, trigger configuration, content delivery, API contracts, and failure handling.

<p align="center">
  <img src="https://img.shields.io/badge/Telegram-Bot_API-26A5E4?style=for-the-badge&logo=telegram&logoColor=white" alt="Telegram">
  <img src="https://img.shields.io/badge/Postman-FF6C37?style=for-the-badge&logo=postman&logoColor=white" alt="Postman">
  <img src="https://img.shields.io/badge/OpenAPI-6BA539?style=for-the-badge&logo=openapiinitiative&logoColor=white" alt="OpenAPI">
  <img src="https://img.shields.io/badge/Chrome_DevTools-4285F4?style=for-the-badge&logo=googlechrome&logoColor=white" alt="Chrome DevTools">
</p>

| Product risk | QA approach | Release decision |
|---|---|---|
| Paid Telegram-bot onboarding and content delivery | AC traceability, API checks, risk matrix, defect evidence and RCA hypotheses | **NO-GO** because the paid onboarding path is blocked |

**Start here:** [test report](docs/TEST_REPORT.md) · [test matrix](test-artifacts/TEST_MATRIX.md) · [evidence index](evidence/README.md)

## Scope

The assessed flow covers creation of a keyword-triggered Telegram chatbot:

```text
paywall
  ↓
bot token
  ↓
channel/admin binding
  ↓
trigger configuration
  ↓
content delivery
```

The objective was to make an evidence-based release decision rather than only validate the happy path.

## QA surface

- paid onboarding and entitlement-dependent flows;
- ChatPlace API contract behavior;
- Telegram Bot API integration boundary;
- channel visibility/binding after bot administrator assignment;
- keyword trigger configuration;
- content-delivery path;
- UI↔API state consistency;
- blocked states and user feedback;
- product-risk prioritization.

## Release result

**NO-GO.**

The main blocking finding affects paid onboarding: `GET /telegram-channels` returns `200 OK` with an empty array even when the bot is confirmed as a channel administrator, while the UI cannot proceed and provides no useful recovery feedback.

The blocker prevents execution of later runtime steps on the same environment. Those scenarios remain designed and traceable but are correctly marked **Blocked** rather than silently reported as passed or omitted.

## Traceability model

The documentation links requirements, execution and risk by stable IDs:

```text
Acceptance Criteria
      ↓
Test Cases
      ↓
Defect Register
      ↓
Risk Matrix
      ↓
Evidence
```

Artifacts use `AC-xx`, `TC-xx`, `BUG-xx` and `OBS-xx` identifiers so a requirement can be followed through to its execution result and supporting evidence.

## Repository map

| Path | Purpose |
|---|---|
| `docs/TEST_REPORT.md` | narrative assessment and Go/No-Go decision |
| `test-artifacts/TEST_MATRIX.md` | AC traceability, test cases, defects, observations and risks |
| `api/*.postman_collection.json` | ChatPlace and Telegram API checks |
| `evidence/` | supporting screenshots and evidence legend |

## QA approach

### Shift-left contract review

UI behavior was compared with API contracts before treating frontend symptoms as isolated UI defects.

### Risk-based testing

The paid onboarding path was prioritized because failure there blocks activation and creates direct funnel/business impact.

### Runtime validation

The intended business path was evaluated as a state transition rather than a set of unrelated screens.

### RCA discipline

Hypotheses were updated as evidence changed. A suspected sync-service failure was not presented as root cause after additional checks moved the likely failure boundary toward API/UI binding and persisted channel state.

### Responsibility boundaries

Findings distinguish ChatPlace behavior from Telegram Bot API dependencies and user-side configuration.

## Test matrix structure

`TEST_MATRIX.md` contains:

1. **AC_Traceability** — requirement ↔ test mapping;
2. **Test_Cases** — Pass / Fail / Blocked execution state;
3. **Defect_Register** — severity, reproduction, evidence and RCA hypothesis;
4. **Observations** — non-blocking technical/product findings;
5. **Risk_Matrix** — probability × impact prioritization.

## Tooling

**Postman · Swagger/OpenAPI · Chrome DevTools Network/Console · Charles Proxy · Telegram / BotFather · Jira-style defect reporting · risk-based release analysis**

## Publication note

This repository documents QA work against a real commercial product. It is presented as a technical QA case and does not claim a paid client engagement where that is not documented.
