# API Testing Portfolio

Regression suite and QA documentation for a public REST API, built with Postman. Includes test strategy, designed test cases, schema validation, a defect log of real findings, and CI execution on GitHub Actions.

## Status

In progress — started September 2026.

## System Under Test

The GitHub REST API, exercised against a dedicated sandbox repository. Chosen over the usual practice APIs because it is a production service with real authentication, rate limiting, pagination and error semantics (401 / 403 / 404 / 422), which makes negative and boundary testing meaningful rather than simulated.

**Base URL:** `https://api.github.com`
**Sandbox repository:** `barblarrosa/api-testing-sandbox` (private — practice data only, not meant to be browsed)

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

```
postman/
  collections/        # Exported Postman collections
  environments/       # Environment templates (no credentials)
  data/               # CSV and JSON files for data-driven testing
  schemas/            # JSON Schema definitions for contract validation
docs/                 # Test strategy, test cases, defect log
reports/              # Curated execution reports (sample runs only)
.github/workflows/    # CI pipeline configuration
```

## Credentials

No tokens or secrets are stored in this repository. Environment templates ship with placeholder values only.

For local development, add two secrets to Postman Vault, scoped to your own sandbox repository:

- `github_token_write` — fine-grained PAT with read/write access to Issues
- `github_token_readonly` — fine-grained PAT with read-only access to Issues and metadata (used for negative authorization tests)

CI secret configuration will be documented once the pipeline is built.
