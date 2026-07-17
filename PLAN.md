# PLAN

## Objective

Build, deploy, directly verify, document, and push a qualifying always-on AWS agent plus a 500+ word Builder Center article before the challenge window becomes crowded.

## Scope and non-goals

### In scope

- **AWS Change Scout**: an hourly agent that reads the official AWS What's New feed, asks Amazon Nova to turn recent announcements into a short action-oriented brief for a fixed personal interest profile (serverless, AI agents, developer tooling, and cost), persists the latest brief, and exposes it on a public proof page.
- Reproducible AWS SAM infrastructure for Lambda plus an explicit CloudFormation `AWS::Events::Rule`, DynamoDB, Bedrock permissions, CloudWatch logs, and a Lambda Function URL. The explicit rule preserves the native Scheduled Event envelope while allowing CloudFormation to resolve enabled/disabled state reliably.
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
- Open and audit a still-empty deployed Function URL on desktop and mobile before any seed or scheduled invocation; confirm the explicit empty state and no console/network failures. If the main table already contains a report from an earlier deployment attempt, use a temporary exact-SHA validation stack with its rule disabled, inspect it, then delete that stack through SAM.
- Redeploy the same SHA with an enabled `rate(1 minute)` explicit EventBridge rule. A manual scheduled-shaped invocation may assist diagnostics but never counts as autonomous proof.
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
- 2026-07-17: Second review added the pushed-SHA deployment order, locked scheduling to a native EventBridge scheduled-rule envelope, and added direct deployed empty-state verification before enabling the schedule. The implementation initially used SAM `Type: Schedule`; later deployed evidence required an explicit `AWS::Events::Rule` to make parameterized state reliable.
- Required workflow gates: load `frontend-patterns` and `frontend-testing` before page implementation/checks; load `refactoring` after green deterministic checks; load `deploy-verify` before deployed verification. The globally referenced `docs` skill is unavailable, so review README/article directly against the live system.
- 2026-07-17: Loaded `frontend-patterns` and `frontend-testing`. After two revisions, the independent plan reviewer returned `PASS`; the mandatory plan-review gate is complete. Final self-review agrees the scope is narrow, the implementation is the simplest robust qualifying path, proof covers autonomous execution plus direct use, and rollback/failure/security cases are explicit. Implementation may start.
- 2026-07-17: Step 2 started: implementing the Lambda schedule/HTTP boundaries, bounded RSS-to-Nova pipeline, safe renderer, and public-behavior tests. Plan remains valid after progress review.
- 2026-07-17: Step 2 implementation complete. Ten behavior-focused tests pass for schedule success/retry/failure, read-only HTTP success/empty/404/405, escaping, event validation, model-output bounds, and feed bounds. Python compilation and `sam build` pass. Step 3 started; initial SAM lint warning was resolved by exposing the conditional schedule state as a useful stack output, and `sam validate --lint` now passes. Plan remains valid after progress review.
- 2026-07-17: Real pre-deploy integration probe fetched the official live feed, parsed eight bounded items, and sent the production `messages-v1` request to Nova Micro; it returned a valid 1,025-character grounded brief. The feed was already ~241 KiB, so review feedback led to a bounded 1 MiB cap with normal release-day headroom.
- 2026-07-17: Step 3 complete. README documents architecture, three-phase exact-SHA deployment, verification, least-privilege/security boundaries, costs/concurrency tradeoff, and teardown. Repeated checks pass: 10 tests, Python compilation, SAM lint/build, whitespace check, and credential-pattern scan.
- Refactoring assessment: fix-now items: none after the feed-bound correction. The single-file agent is long but remains a cohesive Lambda boundary; extra layers would add ceremony to a small challenge build. Existing helpers already centralize security headers, URL validation, bounds, rendering, and model invocation. Leave structure unchanged and avoid cosmetic churn. Plan remains valid after progress review.
- Step 4 started: review, commit, and push the passing implementation to public `main` before deploying that exact SHA.
- 2026-07-17: First disabled-schedule stack creation from pushed SHA `ff51baad5182312b53e64e5fb27c00e011c9ea35` rolled back because this restricted/new-account Lambda quota rejected `ReservedConcurrentExecutions: 1` even though account settings reported 50 total/unreserved. Root-cause fix: remove the optional reservation; the scheduled handler's run-ID/staleness checks and hourly cadence remain sufficient. Rerun all checks, commit/push a new exact SHA, and redeploy after rollback completes. The plan still covers the intended outcomes and gains compatibility without widening scope.
- 2026-07-17: Post-fix rerun passes all 10 tests, Python compilation, SAM lint/build, whitespace check, and credential scan. The README's SAM parameter quoting was also corrected based on the first real CLI invocation. Self-review confirms both edits are root-cause fixes and the deployment/verification plan remains valid.
- 2026-07-17: Pushed corrected SHA `a9332d50cbdc7f6039eacefac8913b37e001bf7b` and created the stack. Live inspection found a second root cause: SAM translated intrinsic `Enabled: !If [...]` as enabled even when the parameter/output said disabled. The one-minute rule produced a genuine report, so autonomous execution works, but it prevented the empty-state audit. The stack was immediately restored to the hourly cadence through SAM. Planned fix: replace only the SAM event shorthand with explicit `AWS::Events::Rule` and `AWS::Lambda::Permission` resources using CloudFormation `State: !If [...]`; the Lambda event envelope and product behavior remain unchanged. Verify the disabled state against the control plane, use a temporary exact-SHA stack for the empty visual state, then re-run autonomous proof on the final main stack. This is a narrower and more reliable solution, with CloudFormation/SAM rollback intact.
- Revised plan review for the explicit-rule correction: independent reviewer returned `PASS`. The correction preserves the native event contract and IaC workflow while making state truthful; implementation may proceed.
- 2026-07-17: Implemented the explicit EventBridge rule plus scoped Lambda permission and added a regression contract proving CloudFormation, not SAM shorthand, resolves schedule state. All 11 tests, Python compilation, SAM lint/build, whitespace check, and credential scan pass. Refactoring reassessment finds no further value-add changes; commit/push/redeploy the new exact SHA next.
- 2026-07-17: Pre-deploy migration review caught that new logical IDs would make CloudFormation add/delete a same-named rule. Renamed the explicit rule and permission to the logical IDs previously generated by SAM so the existing `AWS::Events::Rule` updates in place. This is a deployment-safety correction only; rerun checks and push the final exact SHA before deployment.
- 2026-07-17: Step 4 complete. Passing implementation and infrastructure fixes are committed and pushed to public `main`; final deployed source SHA is `5a17f55b06b58e267092f56074c222fbdae10606`.
- 2026-07-17: Step 5 deployment evidence — main stack `aws-change-scout` is `UPDATE_COMPLETE` in `us-east-1`; Lambda is Active/Successful on Python 3.12 with 256 MB, 60-second timeout, exact final SHA, and final `rate(1 hour)` environment. Public URL: `https://vjbpofzpe4acawbvxybzaeuaqa0pkpog.lambda-url.us-east-1.on.aws/`.
- 2026-07-17: Empty-state evidence — deployed temporary exact-SHA stack `aws-change-scout-empty-check` with its explicit EventBridge rule confirmed `DISABLED` and `rate(1 hour)`. Direct desktop DOM/layout and 390px mobile browser audits showed the semantic "Waiting for the first scheduled run" state, no horizontal overflow, no run button, and no page-attributable console errors. The temporary stack and its empty table were then deleted successfully through SAM; no user data existed to recover.
- 2026-07-17: Autonomous-run evidence — main rule was enabled at `rate(1 minute)` through SAM. Genuine EventBridge event `907e013a-707a-47ee-5d4e-01c5ff588490`, scheduled `2026-07-17T21:32:30Z`, produced exactly one `agent_run_started` and one `agent_run_completed` with the same Lambda request ID, pinned Nova Micro model, eight feed sources, final build SHA, and 1,449 ms duration. No `agent_run_failed` or duplicate completion exists for that selector. DynamoDB persisted that run before the page was read.
- 2026-07-17: Final-state evidence — SAM restored the rule to enabled `rate(1 hour)`; direct `events describe-rule` confirms the control-plane state, and Lambda environment/page agree. Latest retained report is run `e00b8dd9-8a8f-618a-1e4f-1b15ae36b5b3`, generated `2026-07-17T21:36:20Z` from eight sources with build SHA `5a17f55b06b58e267092f56074c222fbdae10606` and Nova Micro.
- 2026-07-17: Direct-use evidence — unauthenticated `GET /` returned 200 in 0.636 s with the final run ID, hourly cadence, SHA prefix, restrictive security headers, and a readable report. `GET /missing` returned 404 in 0.544 s; `POST /` returned 405 in 0.552 s. CloudWatch for the isolated HTTP window showed only 5 ms `report_read` events on GETs, no generation events for 404/405, and DynamoDB remained on the same run. The Lambda resource policy contains both required public URL actions, with `InvokeFunction` restricted by `lambda:InvokedViaFunctionUrl`, plus the source-ARN-scoped EventBridge permission.
- 2026-07-17: Visual/use audit — inspected success on 1440px DOM metrics and 390px rendered mobile viewport with no horizontal overflow, correct heading/status/report/source semantics, and no app-origin console errors or external page resources. Followed the Cost Efficiency source to the official `aws.amazon.com` announcement and confirmed its title. Final proof screenshot saved at `evidence/aws-change-scout-report.png`.
- Steps 5 and 6 complete: deployed empty/success/negative states, autonomous trigger, direct browser use, telemetry, budgets, permissions, persistence, source navigation, rollback, and final hourly state are all evidenced. Step 7 article authoring and final completion reviews are in progress. Plan remains valid after evidence review.
- 2026-07-17: Step 7 complete. `ARTICLE.md` has the exact required title, standalone `#agents`, every required section, live app/repository links, Mermaid architecture, embedded proof screenshot, and 1,441 validated words (well above 500). It is truthful about the build/deployment failures and does not claim Builder Center publication.
- 2026-07-17: Final security reviewer returned `PASS`; credential/history scans, scoped IAM, public read-only separation, outbound/link/prompt/rendering bounds, logs, headers, and teardown have no blocking finding. Accepted residual risks are documented public-read cost exposure and non-transactional latest-item idempotency in this low-cost disposable proof.
- 2026-07-17: Final maintainability review found no source, infrastructure, documentation, teardown, schedule, or operability blocker. Final completion review found the live app, article, tests, SAM template, screenshot, links, and evidence qualifying and returned a conditional pass whose only expected release blocker is committing/pushing the finished article, screenshot, live README, and final PLAN evidence.
- Step 8 started: commit and push the completed submission artifacts, verify `origin/main` and public GitHub content, then record the final release evidence. Plan remains valid after final review.
- 2026-07-17: Step 8 complete. Commit `8be93cc5b1e91fe30a6203f26fad1b9d3229fd3f` pushed the article, proof image, live README, and accumulated evidence to `origin/main`; local HEAD and remote ref match. An unauthenticated GitHub API check confirms the repository is public with default branch `main`. Unauthenticated raw reads confirm README contains the live Function URL, ARTICLE contains the exact title/tag/link section, and the proof PNG returns HTTP 200 (228,623 bytes).
- Final completion review: all conditional release blockers and the maintainability P0 are addressed. Required deterministic checks pass; the exact final code SHA is deployed and directly verified; telemetry is isolated and within budgets; the final rule is enabled hourly; security review passes; the public repository contains all submission artifacts. The only remaining action is the explicitly out-of-scope human handoff to paste/publish `ARTICLE.md` in AWS Builder Center during the challenge window and apply the platform tag. Task implementation is complete.
