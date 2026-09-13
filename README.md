# QA Test Assignment · ChatPlace · “Keyword Chatbot” (Telegram)

**Candidate:** Vsevolod Samoylov · **Position:** QA Engineer · **Date:** 30.07.2026  
**AI used during preparation:** Qwen for structure and wording; scope, prioritization, Risk Matrix, and Go/No-Go decisions were my own.

<p align="center">
  <img src="https://img.shields.io/badge/Telegram-Bot_API-26A5E4?style=for-the-badge&logo=telegram&logoColor=white" alt="Telegram">
  <img src="https://img.shields.io/badge/Postman-FF6C37?style=for-the-badge&logo=postman&logoColor=white" alt="Postman">
  <img src="https://img.shields.io/badge/OpenAPI-6BA539?style=for-the-badge&logo=openapiinitiative&logoColor=white" alt="OpenAPI">
  <img src="https://img.shields.io/badge/Chrome_DevTools-4285F4?style=for-the-badge&logo=googlechrome&logoColor=white" alt="Chrome DevTools">
</p>

| Product risk | QA approach | Decision |
|---|---|---|
| Paid Telegram-bot onboarding and content delivery | AC traceability, API checks, risk matrix, defect evidence and RCA hypotheses | **NO-GO** because the paid onboarding path is blocked |

**Start here:** [test report](docs/TEST_REPORT.md) · [test matrix](test-artifacts/TEST_MATRIX.md) · [evidence index](evidence/README.md)

## What I tested
The paid flow for creating a keyword-triggered Telegram chatbot:
paywall → token input → bot-admin channel binding → trigger setup → content delivery.

Testing was performed from a release-gatekeeper perspective. The goal was not to click through a happy path, but to make an evidence-based production-readiness decision.

## How to read the repository
| File / folder | Purpose | Recommended order |
|---|---|---|
| `docs/TEST_REPORT.md` | Narrative report + Go/No-Go release decision | first |
| `test-artifacts/TEST_MATRIX.md` | Registry of AC, cases, defects, observations, and risks | data and traceability |
| `api/*.postman_collection.json` | Contract checks for ChatPlace API + Telegram Bot API | artifact package |
| `evidence/` | Supporting screenshots and evidence legend (`evidence/README.md`) | defect attachments |

## `TEST_MATRIX.md` sections
1. `AC_Traceability` — acceptance criteria ↔ test cases.
2. `Test_Cases` — checklist with Pass / Fail / Blocked statuses.
3. `Defect_Register` — bug reports with severity, steps, RCA hypothesis, and evidence.
4. `Observations` — technical debt and non-critical findings.
5. `Risk_Matrix` — probability × impact for each AC, used for risk-based prioritization.

## Traceability
AC ↔ Test_Cases ↔ Defect_Register ↔ Risk_Matrix ↔ evidence are connected by IDs (`AC-xx`, `TC-xx`, `BUG-xx` / `OBS-xx`). This provides end-to-end traceability from requirement to evidence, similar to a TMS workflow in Allure TestOps or Zephyr.

## Release status
**NO-GO.** Blocking defect `BUG-01` affects paid onboarding: `GET /telegram-channels` returns `200 OK` with an empty array even when the bot is confirmed as a channel administrator, while the UI blocks progress without feedback. Details are in section 6 of `docs/TEST_REPORT.md`.

## Approach — short version
1. **Shift-left** — review API↔UI contracts before UI execution.
2. **Risk-based testing** — focus on paid onboarding, the most expensive point in the funnel.
3. **Runtime validation** — verify the “keyword → content delivery” behavior.
4. **RCA** — use a 5 Whys loop: the “dead sync service” hypothesis was disproved by action, and the failure point moved to API↔UI contract / database binding.
5. **Responsibility boundaries** — separate what belongs to ChatPlace from Telegram Bot API and user-side dependencies.

> Execution of steps 2–3 on the environment was **blocked** by `BUG-01`. The related cases were still designed from product requirements and correctly marked *Blocked*, not *Not done*: test design does not depend on environment availability.

## Tools
Postman · Swagger/OpenAPI · DevTools (Network/Console) · Charles Proxy · Kibana · Telegram + @BotFather · Jira · Allure TestOps · GitLab CI.