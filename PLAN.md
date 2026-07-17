# PLAN

## Objective

Build, deploy, directly verify, document, and push a qualifying always-on AWS agent plus a 500+ word Builder Center article before the challenge window becomes crowded.

## Scope and non-goals

### In scope

- **AWS Change Scout**: an hourly agent that reads the official AWS What's New feed, asks Amazon Nova to turn recent announcements into a short action-oriented brief for a fixed personal interest profile (serverless, AI agents, developer tooling, and cost), persists the latest brief, and exposes it on a public proof page.
- Reproducible AWS SAM infrastructure for Lambda, an EventBridge scheduled rule (SAM `Type: Schedule`), DynamoDB, Bedrock permissions, CloudWatch logs, and a Lambda Function URL.
- A small deterministic Python test suite, deployment helper commands, a concise README, direct deployed verification, runtime-log verification, and an article meeting every stated challenge requirement.
- A genuine one-minute scheduled run during verification so the public page becomes useful without a button click, followed by restoration to the hourly cadence.

### Non-goals

- Authentication, personalization, a multi-user product, historical analytics, email delivery, a custom domain, or a polished application framework.
- Supporting arbitrary feeds or user-supplied prompts.
- Building beyond what is needed for a clear, working challenge submission.

## Assumptions and open questions

- Resolved for now: the supplied temporary AWS session authenticated successfully to account `064822870603`; re-check immediately before deployment and request fresh credentials only if it expires.
- Resolved: `amazon.nova-micro-v1:0` is directly invokable in `us-east-1`; a minimal real invocation returned `probe-ok`. The US cross-region profile in `us-west-2` is unusable because an organization SCP explicitly denies one destination, so the stack will stay in `us-east-1`.
- Resolved: `https://github.com/guilleojeda/builder-center-always-on-agent-challenge` is public, empty, and writable by the authenticated GitHub user.
- Public Lambda Function URLs are permitted by account policy; resolve through deployment and an unauthenticated GET. If blocked, switch the proof page to an S3 static site without changing the agent core.

## Expected outcomes

### Product design

AWS Change Scout is for a builder focused on serverless, AI agents, developer tooling, and cost who wants to keep up with AWS releases without repeatedly scanning the announcements feed. The ideal experience is passive: EventBridge wakes the agent each hour, the agent fetches only the allow-listed official feed, Nova identifies why the most relevant changes matter and recommends a next action, and DynamoDB makes the result ready to read later. Opening the public proof URL shows the latest generated brief, authoritative source links taken from the parsed feed rather than the model, generation time, schedule, and run identifier. Before the first successful run, the page clearly says that no report exists yet; after a failed refresh, the previous successful report remains available while CloudWatch records and re-raises the failure.

### Specification

#### Scenario: autonomous scheduled run

Given a valid EventBridge scheduled event and reachable AWS feed, when Lambda runs, then it fetches a bounded set of recent announcements, invokes Nova with feed content treated strictly as untrusted data, validates a non-empty bounded plain-text response, stores one latest report, and emits structured completion telemetry keyed by the EventBridge event ID and Lambda request ID.

#### Scenario: public report read

Given a stored report, when an unauthenticated visitor opens the Function URL with `GET /`, then Lambda returns a readable HTML page containing the escaped report, feed-derived source links, timestamp, and run ID without invoking Bedrock or mutating state. Unsupported paths return 404 and unsupported methods return 405 before any model or write path.

#### Scenario: upstream or model failure

Given an upstream, parsing, empty-feed, malformed-model-output, or Bedrock failure, when a scheduled run executes, then it emits `agent_run_failed`, re-raises so AWS records a failed invocation, does not overwrite the last good report, and does not present fabricated AI output as a success.

### Required side effects

- One latest-report item is written to DynamoDB after each successful generation.
- Structured CloudWatch logs identify run start, completion, duration, and failure.
- EventBridge invokes the function automatically at the declared cadence and supplies the schedule event ID/time for correlation.

### Forbidden outcomes

- AWS credentials or tokens in source, git history, logs, output, or article.
- Arbitrary outbound URLs, raw feed HTML rendered into the page, prompt instructions from feed content being followed, or a successful status when Bedrock did not produce the report.
- A public GET triggering a billable model run.
- A DynamoDB write when fetching, parsing, model invocation, or model-output validation is incomplete.

## Deterministic checks

- `python3 -m unittest discover -s tests -v`: prove the scheduled handler stores a Nova-backed report, GET renders stored content safely with security headers, missing reports render an explicit empty state, 404/405 requests cannot reach generation, untrusted feed/model text is escaped, bounded parsing/output contracts hold, malformed events are rejected, and failed generations preserve the previous report while re-raising.
- `python3 -m compileall src tests`: prove Python syntax/importability.
- `sam validate --lint`: prove the infrastructure template is valid.
- `sam build`: prove the Lambda artifact builds with the declared runtime.
- Secret scan with `git grep` patterns and repository inspection before commit/push.
- Article validation script/check: title phrase, `#agents`, required headings, repository/deployed link, and at least 500 words.

## Deployed verification

- Commit and push passing source/template/tests to `main`, then deploy that exact git SHA in `us-east-1` with the schedule initially disabled, a SHA stack parameter/environment value, and `amazon.nova-micro-v1:0`.
- Open and audit the still-empty deployed Function URL on desktop and mobile before any seed or scheduled invocation; confirm the explicit empty state and no console/network failures.
- Redeploy the same SHA with an enabled `rate(1 minute)` SAM `Type: Schedule` EventBridge rule. A manual scheduled-shaped invocation may assist diagnostics but never counts as autonomous proof.
- Wait for a genuine EventBridge invocation, then correlate its event ID, CloudWatch completion log, DynamoDB item, and public report. Capture the enabled rule and rendered report as repository screenshots/evidence.
- Open the unauthenticated Function URL directly, inspect the rendered desktop and narrow/mobile layouts, follow at least one source link, and confirm timestamp/run ID/report content. Inspect browser console and network failures.
- Call the URL with an unsupported method/path and confirm 404/405 without a model invocation. Error presentation is proved locally because inducing a deployed failure would add a risky test-only control.
- Redeploy the committed `rate(1 hour)` schedule, verify the final rule is enabled with that expression, verify the retained report and deployed source SHA are still public, and inspect the Function URL resource policy for the permissions required by public access.
- If deployed testing requires an implementation/template change, rerun deterministic checks, commit/push it, redeploy that new exact SHA, and repeat direct plus telemetry verification. Article/screenshots may follow in a documentation-only commit without invalidating the deployed code SHA.

## Telemetry verification

### Selectors

- CloudFormation stack name and deployed Lambda log group.
- Unique `run_id` from the validation invocation and public report.
- Deployed Lambda version/configuration timestamp and exact pushed git SHA exposed by the page/logs.

### Required signals

- One `agent_run_started` and one `agent_run_completed` record for the isolated run.
- A successful in-region Bedrock invocation, positive bounded source-item count, DynamoDB persistence, and duration within the Lambda timeout.
- A subsequent public GET returns HTTP 200 and the same run ID without a model invocation.

### Forbidden signals

- `agent_run_failed`, Lambda timeout, unhandled exception, credential material, or duplicate completion for the isolated run.
- Bedrock invocation or DynamoDB write during GET, 404, or 405 requests.

### Budgets

- Scheduled generation completes within 60 seconds.
- Public GET completes within 3 seconds under normal warm/cold Lambda behavior.

## Implementation steps

1. Confirm repository, AWS, Bedrock, and GitHub constraints and complete the reviewed plan.
   - Done when: the plan-review gate passes and deployment assumptions have a clear resolution path.
2. Add the minimal agent, public report renderer, and high-signal tests.
   - Done when: the scheduled/public/error behaviors pass locally without real AWS calls.
3. Add the SAM template and operator documentation.
   - Done when: SAM validation/build and documentation checks pass.
4. Commit and push the passing implementation to `main`.
   - Done when: source, template, tests, and operator docs are on the public remote and the exact git SHA is ready for deployment.
5. Deploy and directly verify AWS Change Scout from that exact SHA.
   - Done when: the disabled-schedule empty state is audited, the enabled rule autonomously creates a real Nova-backed report, and the stack exposes the deployed SHA.
6. Directly verify the schedule, public page, failure boundary, and telemetry.
   - Done when: all planned observable evidence is recorded, including an actual autonomous invocation, desktop/mobile browser audit, console/network inspection, and screenshots.
7. Write and validate the challenge article and visual evidence.
   - Done when: `ARTICLE.md` uses the exact title `Weekend Agent Challenge: AWS Change Scout`, includes standalone `#agents`, every required heading, at least 700 prose words, the live Function URL, public repository URL, architecture diagram, and scheduled-run/report screenshots.
8. Review, commit, push, and verify the public repository.
   - Done when: required reviews are addressed, relevant files are committed on `main`, push succeeds, and the remote content is reachable.

## Risks and constraints

- Temporary credentials may expire; minimize sequential work and deploy once the deterministic checks pass.
- Bedrock access is pinned to the successfully probed in-region Nova Micro model. Nova Micro returns plain text rather than structured output; require non-empty output, cap output length/tokens, and reject invalid results.
- RSS content is untrusted; use one fixed HTTPS URL, a short timeout, byte/item/title/description bounds, XML parsing, HTML escaping, and an explicit data-only prompt boundary. Feed-derived links are validated HTTPS AWS URLs and the model never supplies links.
- Public Function URLs can be abused; route HTTP before schedule handling, allow only read-only `GET /`, add basic browser security headers, and never invoke Bedrock from HTTP. SAM must provision and deployed inspection must confirm both public Function URL permissions required by current AWS behavior.
- Lambda IAM is limited to `GetItem`/`PutItem` on one table, logs, and `InvokeModel` on the selected Nova resource.
- The hourly schedule has small ongoing cost. The README must include teardown instructions because the account is temporary.
- The verification schedule is a temporary one-minute configuration; restoring and verifying the committed hourly schedule is the rollback and a completion condition. `sam delete` must remove the schedule, URL, function, and table.
- The initially disabled schedule is only for directly auditing the deployed empty state. Enabling it changes stack parameters, not source; every deployed stack identifies the exact pushed source SHA.
- The task prioritizes qualifying speed; avoid optional UI, notification, and history features.
- Builder Center publication/submission is a user handoff because no authenticated publication surface was supplied; completion must not claim the contest entry itself was submitted.

## Progress, blockers, and evidence

- 2026-07-17: Classified as delivery work. Loaded `planning`, `specification`, `testing`, `backend-patterns`, and `security` skills. Repository is initially empty with a GitHub `origin` and no commits.
- 2026-07-17: Initial AWS CLI identity check used stale ambient credentials; supplied temporary credentials still require explicit validation without persisting them in the repository.
- 2026-07-17: Supplied session validated as account `064822870603`. A real `amazon.nova-micro-v1:0` invocation succeeded in `us-east-1`; the `us-west-2` cross-region profile failed due an organization SCP, so `us-east-1` is selected.
- 2026-07-17: Independent plan review found nine material gaps. All are incorporated: autonomous-run correlation, pinned Bedrock path, bounded plain-text contract, current Function URL permissions, qualification-ready article/evidence, failure semantics, frontend/refactoring/deploy gates, least privilege/rollback, and personal-interest ranking.
- 2026-07-17: Second review added the pushed-SHA deployment order, explicitly locked the SAM event to `Type: Schedule`, and added direct deployed empty-state verification before enabling the schedule.
- Required workflow gates: load `frontend-patterns` and `frontend-testing` before page implementation/checks; load `refactoring` after green deterministic checks; load `deploy-verify` before deployed verification. The globally referenced `docs` skill is unavailable, so review README/article directly against the live system.
- 2026-07-17: Loaded `frontend-patterns` and `frontend-testing`. After two revisions, the independent plan reviewer returned `PASS`; the mandatory plan-review gate is complete. Final self-review agrees the scope is narrow, the implementation is the simplest robust qualifying path, proof covers autonomous execution plus direct use, and rollback/failure/security cases are explicit. Implementation may start.
- 2026-07-17: Step 2 started: implementing the Lambda schedule/HTTP boundaries, bounded RSS-to-Nova pipeline, safe renderer, and public-behavior tests. Plan remains valid after progress review.
- 2026-07-17: Step 2 implementation complete. Ten behavior-focused tests pass for schedule success/retry/failure, read-only HTTP success/empty/404/405, escaping, event validation, model-output bounds, and feed bounds. Python compilation and `sam build` pass. Step 3 started; initial SAM lint warning was resolved by exposing the conditional schedule state as a useful stack output, and `sam validate --lint` now passes. Plan remains valid after progress review.
- 2026-07-17: Real pre-deploy integration probe fetched the official live feed, parsed eight bounded items, and sent the production `messages-v1` request to Nova Micro; it returned a valid 1,025-character grounded brief. The feed was already ~241 KiB, so review feedback led to a bounded 1 MiB cap with normal release-day headroom. Lambda quota preflight shows 50 total/unreserved concurrency, sufficient for the function's reservation of one.
- 2026-07-17: Step 3 complete. README documents architecture, three-phase exact-SHA deployment, verification, least-privilege/security boundaries, costs/concurrency tradeoff, and teardown. Repeated checks pass: 10 tests, Python compilation, SAM lint/build, whitespace check, and credential-pattern scan.
- Refactoring assessment: fix-now items: none after the feed-bound correction. The single-file agent is long but remains a cohesive Lambda boundary; extra layers would add ceremony to a small challenge build. Existing helpers already centralize security headers, URL validation, bounds, rendering, and model invocation. Leave structure unchanged and avoid cosmetic churn. Plan remains valid after progress review.
- Step 4 started: review, commit, and push the passing implementation to public `main` before deploying that exact SHA.
