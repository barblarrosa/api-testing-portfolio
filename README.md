# API Testing Portfolio

Regression suite and QA documentation for a public REST API, built with Postman.
Includes test strategy, designed test cases, schema validation, a defect log of
real findings, and CI execution on GitHub Actions.

## Status

In progress — started September 2026.

## System Under Test

The GitHub REST API, exercised against a dedicated sandbox repository.
Chosen over the usual practice APIs because it is a production service with real
authentication, rate limiting, pagination and error semantics (401 / 403 / 404 /
422), which makes negative and boundary testing meaningful rather than simulated.

Base URL: `https://api.github.com`

## Scope

- Functional coverage of issues, labels and gists (full CRUD lifecycle)
- Authentication and authorization behaviour
- Contract validation against JSON Schema
- Data-driven execution over multiple input sets
- Error handling, rate limit headers and pagination

## Stack

- Postman — collections, environments, scripting and assertions
- JavaScript — test scripts (`pm.*` API, current syntax)
- JSON Schema — contract validation
- Postman CLI — execution in CI
- Newman + htmlextra — HTML execution reports
- GitHub Actions — scheduled and on-demand runs

## Repository Structure

- `collections/` — exported Postman collections
- `environments/` — environment templates (no credentials)
- `data/` — CSV and JSON files for data-driven testing
- `schemas/` — JSON Schema definitions used for contract validation
- `docs/` — test strategy, test cases, defect log, traceability matrix
- `reports/` — execution reports
- `.github/workflows/` — CI pipeline

## Credentials

No tokens or secrets are stored in this repository. Environment files are
committed as templates with empty values; secrets are resolved from Postman Vault
locally and from repository secrets in CI.

## Author

Barbara Larrosa — ATS Analyst
[LinkedIn](https://www.linkedin.com/in/barbaralarrosa)
