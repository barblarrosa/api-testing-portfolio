# API Testing Portfolio

Regression suite and QA documentation for a public REST API, built with Postman. Includes test strategy, designed test cases, schema validation, a defect log of real findings, and CI execution on GitHub Actions.

## Status

In progress — started September 2026.

## System Under Test

The GitHub REST API, exercised against a dedicated sandbox repository. Chosen over the usual practice APIs because it is a production service with real authentication, rate limiting, pagination and error semantics (401 / 403 / 404 / 422), which makes negative and boundary testing meaningful rather than simulated.

Base URL: `[https://api.github.com](https://api.github.com)`

## Scope

- Functional coverage of issues and labels (full CRUD lifecycle)
- Authentication and authorization behaviour (401 Unauthorized vs 403 Forbidden)
- Contract validation against JSON Schema
- Data-driven execution over multiple input sets
- Error handling, and rate limit headers

## Stack

- **Postman** — collections, environments, scripting and assertions
- **JavaScript** — test scripts (`pm.*` API, current syntax)
- **JSON Schema** — contract validation
- **Newman + htmlextra** — command-line execution and rich HTML reports
- **GitHub Actions** — CI pipeline (event-triggered and manual runs)

## Repository Structure
├── collections/        # Exported Postman collections
├── environments/       # Environment templates (no credentials)
├── data/               # CSV and JSON files for data-driven testing
├── schemas/            # JSON Schema definitions used for contract validation
├── docs/               # Test strategy, test cases, defect log
├── reports/            # Curated execution reports
└── .github/workflows/  # CI pipeline configuration

## Credentials

No tokens or secrets are stored in this repository. Environment files are committed as templates with empty values; secrets are resolved from Postman Vault locally and from repository secrets (`SANDBOX_PAT`) in CI.

## Author

Barbara Larrosa — ATS Analyst · [LinkedIn](https://www.linkedin.com/in/barblarrosa/)
