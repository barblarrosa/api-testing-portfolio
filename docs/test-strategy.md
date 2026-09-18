# Test Strategy

## 1. Purpose

This document explains how the API regression suite in this repository was designed:
which risks it targets, which test types cover them, what is deliberately out of
scope, and what a passing run means.

It is written to answer one question: why these tests and not others.

## 2. System under test

The target is the GitHub REST API, limited to the Issues and Labels resources,
exercised against a sandbox repository owned by the author.

| Item | Value |
| --- | --- |
| Base URL | `https://api.github.com` |
| Target repository | `barblarrosa/api-testing-sandbox` (public) |
| API version header | `X-GitHub-Api-Version: 2022-11-28` |
| Accept header | `application/vnd.github+json` |
| Authentication | Fine-grained personal access token |

GitHub was chosen over a mock or locally hosted API because it forces the suite to
deal with conditions a toy API does not have: real token-based authorization,
permission scopes, pagination, rate limiting, and a published contract that can be
tested against observed behavior.

### Characteristics that shape the design

Four properties of this API drive most of the decisions below.

**Issues cannot be deleted through the REST API.** The API exposes no delete
endpoint for issues; an issue is retired by setting its state to `closed`.
Deletion exists only in the GraphQL API. Labels, by contrast, support a full
create-read-update-delete lifecycle. The suite therefore manages two resources
with different lifecycles and needs two different teardown paths.

**Issue endpoints return pull requests.** GitHub models every pull request as an
issue, so list responses can contain both. Pull requests are identifiable by the
`pull_request` key. Any assertion that counts or filters issues must account for
this or it will be wrong the first time a pull request exists in the repository.

**The repository is public.** Read requests succeed without credentials, so an
unauthenticated request returns data rather than an authentication error. This
removes one negative case and adds a different one: unauthenticated traffic is
capped at 60 requests per hour against 5,000 for an authenticated token, which is
observable in the rate limit response headers.

**Creating content is rate limited beyond the primary quota.** GitHub applies
secondary rate limits to content creation and to bursts against a single endpoint.
A suite that creates issues in a tight loop will trigger them. This constrains
data volume, not just execution speed.

## 3. Scope

### In scope

- Issues: create, read, list, update, close, label assignment.
- Labels: create, read, list, update, delete.
- Cross-cutting behavior on both resources: authentication and authorization,
  response schema, error responses, pagination, and required headers.

### Out of scope, with rationale

| Excluded | Rationale |
| --- | --- |
| Issue comments | Adds no technique not already covered. Request chaining is exercised by issues and labels, so comments would add endpoints to maintain without adding coverage. |
| Gists | Account-level resources. A failed teardown would leave artifacts on the author's public profile rather than inside a disposable sandbox. |
| Pull requests | Out of scope as a resource, but their presence in issue list responses is treated as a data condition to be handled. |
| Repository administration, webhooks, Actions API | Not required to demonstrate the techniques in scope; each would widen the token permissions needed. |
| Performance and load testing | Deliberately excluded. Sustained load against a third-party API would breach its rate limits and terms of use. Response time is asserted only as a coarse sanity threshold, not as a performance measurement. |
| Security testing beyond authorization | Injection, fuzzing, and similar techniques are not appropriate against a third-party production service. |

## 4. Test approach

### Risk-based selection

Tests are selected by risk, not by endpoint count. A request is only worth a test
if a plausible failure in it would be noticed by a consumer of the API. Section 5
maps every risk to the coverage that addresses it; anything in the suite that does
not trace back to a row in that table does not belong there.

### Assertion standard

A status code alone proves almost nothing. A server can return `201 Created` and
persist the wrong data, or return the correct body and fail to store it. Every
write in this suite is therefore verified by read-after-write: the response is
asserted against the payload that was sent, and an independent GET confirms the
resource persisted with those values.

Every assertion must name the behavior it protects. If the behavior that would
break it cannot be stated in one sentence, the assertion is removed.

### Coverage types

| Type | What it covers here |
| --- | --- |
| Positive | Documented happy paths for both resources, including read-after-write verification. |
| Negative | Malformed and invalid payloads, missing required fields, unknown enum values, non-existent resources. |
| Boundary | Field length limits, pagination limits, empty and single-element collections. |
| Contract | JSON schema validation of response bodies, field types, and required keys. |
| Authorization | Invalid credentials, insufficient permissions, and the distinction between them. |
| Error handling | Status code correctness and consistency of the error response shape across endpoints. |

### Authorization model under test

The three failure codes are distinct and each is tested separately, because
conflating them is the most common error in API test suites.

| Code | Condition | Trigger used |
| --- | --- | --- |
| 401 | Credential is absent or unusable | Request sent with an invalid bearer token |
| 403 | Credential is valid but lacks the permission for the operation | Write attempted with a read-only permission scope |
| 404 | Resource does not exist, or exists but is invisible to the credential | Request for a non-existent issue number |

GitHub returns 404 rather than 403 for resources a credential cannot see at all,
so that response codes do not disclose the existence of private resources. The
suite asserts that behavior rather than treating it as an anomaly.

## 5. Risk analysis

| ID | Risk | Impact if unaddressed | Coverage |
| --- | --- | --- | --- |
| R1 | The token can perform operations beyond its declared permission scope | Privilege escalation; a least-privilege claim that is not true | Authorization tests for 401, 403, and 404 |
| R2 | Response contract changes: a field disappears or changes type | Silent breakage in every client that parses the response | JSON schema validation on all read and write responses |
| R3 | False persistence: a success status is returned but state is not stored as requested | Data loss that no status-code assertion would detect | Read-after-write verification on every create and update |
| R4 | Input validation is weaker than documented | Invalid data accepted into the system | Negative and boundary tests on required fields, field length, and enum values |
| R5 | Error responses are inconsistent in shape or status across endpoints | Clients cannot handle errors generically | Error contract assertions on status, `message`, and error body structure |
| R6 | Test execution leaves artifacts in the repository | A public repository filling with test data; later runs polluted by earlier ones | Naming convention, teardown requests, and a sweep that closes orphaned artifacts |
| R7 | Rate limiting makes runs non-deterministic | A red CI badge caused by the suite, not by the API | Bounded data volume, no retry loops, and assertions on rate limit headers |

## 6. Test data and state management

All test data is generated at run time. No data is assumed to pre-exist in the
sandbox beyond the repository itself, and no test depends on an artifact created by
a previous run.

Because the sandbox is public, every artifact the suite creates is visible. Issue
titles and label names therefore carry a fixed prefix plus a per-run identifier, so
that anything the suite creates is immediately identifiable as automation output.

Teardown is asymmetric by necessity:

- **Labels** are deleted. The delete endpoint returns `204 No Content`.
- **Issues** are closed, because the REST API provides no way to delete them. The
  suite asserts the transition to `closed` rather than asserting a 404 afterwards,
  since the resource legitimately still exists.

A run that fails partway through will leave artifacts behind. That is mitigated by
a cleanup step that finds open artifacts matching the naming convention and closes
or deletes them, so state accumulated by an earlier failure does not affect the
next run.

## 7. Environment and authentication

A single environment is used: the sandbox repository.

Authentication uses a fine-grained personal access token restricted to that one
repository, with Issues read and write and Metadata read. Nothing broader is
granted. The 403 case is produced by a second token with read-only permission on
Issues.

Tokens are never committed and never written into an exported environment file.
Locally they are held in Postman Vault. In CI the token is supplied from a
repository secret named `SANDBOX_PAT` and injected at run time. The secret is not
named with a `GITHUB_` prefix because that namespace is reserved by GitHub Actions.

The environment file committed to this repository is a template containing keys
with empty values. Its purpose is to let anyone clone the repository and run the
suite with their own sandbox and their own token.

## 8. Execution model

The suite is designed to run identically in two places.

**Locally**, through the Postman collection runner, with the token resolved from
Postman Vault. This is where tests are written and debugged.

**In CI**, through GitHub Actions running Newman, on push, on pull request, and on
manual dispatch. The token is passed as an environment variable from the repository
secret. A report is produced with newman-reporter-htmlextra and published as a
workflow artifact.

The report generation explicitly omits the Authorization header, because the
reporter includes request headers by default and would otherwise publish the token
inside a downloadable artifact.

Scheduled runs are not configured. A fine-grained token expires, and a schedule
would turn every expiry into a failing badge with no one watching.

## 9. Entry and exit criteria

### Entry criteria

- The sandbox repository exists and has Issues enabled. If Issues are disabled the
  API returns `410 Gone` and the entire suite is invalid.
- A token with the required permissions is available to the runner.
- Collection and environment variables resolve; no request depends on a hardcoded
  identifier.
- The suite is idempotent: it can run twice in a row without manual cleanup
  between runs.

### Exit criteria

A run is acceptable when:

- Every request executes and every assertion either passes or fails for a reason
  recorded in the defect log.
- No artifact created during the run remains open or undeleted.
- The remaining rate limit at the end of the run is high enough that an immediate
  second run would succeed.
- A report is produced and contains no credential material.

## 10. Known constraints

- **Rate limits.** 5,000 requests per hour for an authenticated token, 60 for
  unauthenticated requests, plus secondary limits on content creation and on bursts
  against a single endpoint. The suite is sized well below these and does not retry.
- **Notifications.** Creating issues generates notifications on the account that
  owns the sandbox. This is expected, not a defect.
- **Third-party dependency.** The API can change without notice. A failure may
  reflect a genuine change in behavior rather than a regression in the suite, so
  every failure is triaged before it is logged.
- **Public artifacts.** Everything the suite creates is publicly visible while it
  exists. This is the reason for the naming convention and the cleanup sweep.
- **Vault is local only.** Postman Vault does not resolve under Newman, so the CI
  path injects the token separately. The collection must not assume either source.

## 11. Defect reporting

Findings are recorded in `docs/defect-log.md`. Only findings reproduced by an actual
execution are recorded; nothing is written from expectation alone.

Each finding is classified:

| Class | Definition |
| --- | --- |
| Functional | Observed behavior contradicts the documented behavior of the endpoint. |
| Contract | Response structure or field type deviates from the documented schema. |
| Documentation | The API behaves consistently, but the documentation describes it incorrectly or incompletely. |
| Error handling | Status code or error body is inconsistent with comparable endpoints. |
| Not a defect | Reproducible behavior that is correct but non-obvious. Recorded separately as an observation, not as a bug. |

A test failure caused by an error in the test itself is corrected, not logged.
Distinguishing the two is part of the exercise, and mislabeling a test defect as a
product defect is treated as a worse outcome than finding nothing at all.
