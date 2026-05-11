# actions — Repository context

> **Onboarding handshake:** Read in this order:
>
> 1. `Projects/CLAUDE.md` (global standards, workspace-local)
> 2. `tcwlab/CLAUDE.md` (toolchain context, consumer API, workspace-local)
> 3. This file (actions-specific details)

---

## What is `actions`?

`actions` is the repo where tcwlab's own **Forgejo composite actions** live. Composite actions are YAML building blocks that bundle multiple steps into a reusable, parameterizable unit with inputs and outputs. Consumer repos reference them via `uses:` in their workflows:

```yaml
- uses: https://<your-forgejo-host>/tcwlab/actions/.forgejo/actions/<name>@<ref>
  with:
    input1: value1
```

Together with `templates/`, this repo forms the **consumer API** of the entire toolchain. While `templates/` provides full workflow files (for drop-in copy), composite actions provide finer-grained building blocks that can be embedded in any workflow.

### Status

🟡 **First action live: `checkout-shell`.** Reusable for any tcwlab consumer that runs in a container without Node.js. Released as `v1` / `v1.0.0`. More candidates (`trivy-scan-pr-comment`, `semantic-release-forgejo`, `docker-buildx-publish`, `betterlint-with-pr-comment`) follow as patterns get extracted from consumer repos.

### Consumers

All TCW repos that want to bundle recurring CI logic. Primary candidates for the first composite actions:

- **`trivy-scan-pr-comment`** — Trivy scan plus Markdown report plus PR description update via Forgejo API. This logic already exists in multiple image repos and should be centralized.
- **`semantic-release-forgejo`** — semantic-release step including Forgejo authentication variables and `.NEXT_RELEASE_VERSION` output.
- **`docker-buildx-publish`** — Multi-architecture build + push to Docker Hub (and eventually Harbor) with auto-tagging.
- **`betterlint-with-pr-comment`** — Lint run plus optional PR comment on findings.

---

## What's in the repo?

The structure (planned for when the first actions are implemented):

```text
actions/
├── CLAUDE.md                                ← this file
├── README.md                                ← consumer API documentation
├── .forgejo/
│   └── actions/
│       ├── trivy-scan-pr-comment/
│       │   ├── action.yml                   ← composite action definition
│       │   └── README.md                    ← inputs, outputs, examples
│       ├── semantic-release-forgejo/
│       │   ├── action.yml
│       │   └── README.md
│       ├── docker-buildx-publish/
│       │   ├── action.yml
│       │   └── README.md
│       └── betterlint-with-pr-comment/
│           ├── action.yml
│           └── README.md
└── .forgejo/workflows/
    └── ci.yml                               ← lint + (eventually action tests)
```

`action.yml` follows the Forgejo/GitHub composite action spec: `name`, `description`, `inputs:`, `outputs:`, `runs.using: composite`, `runs.steps: [...]`.

Each composite action gets its own README documenting inputs, outputs, and usage examples.

---

## Versioning

Composite actions are not Docker-image-based but Git-tag-based. Consumers pin via the Git ref:

```yaml
# Major tag (moves with patches/minors)
- uses: https://<your-forgejo-host>/tcwlab/actions/.forgejo/actions/trivy-scan-pr-comment@v1

# Concrete SemVer
- uses: https://<your-forgejo-host>/tcwlab/actions/.forgejo/actions/trivy-scan-pr-comment@v1.2.0

# SHA (immutable, for audit-required repos)
- uses: https://<your-forgejo-host>/tcwlab/actions/.forgejo/actions/trivy-scan-pr-comment@<sha>
```

semantic-release will version the repo as a whole (`v1.0.0`, `v1.1.0`). Major tags (`v1`) will roll forward to the latest compatible release. This pattern matches the GitHub Actions Marketplace standard.

---

## Release procedure

Like all other `tcwlab` repos: semantic-release in the `release` job, Conventional Commits → SemVer, Forgejo tag, rolling major tag via auto-tag action in the release job. Functionally, we're not building an image — the output is only a Git tag.

---

## What to do when adding a new composite action

1. **PR on `claude/feat-<action-name>`**: New subfolder under `.forgejo/actions/<name>/` with `action.yml` plus `README.md`.
2. **Local verification**: Test in a sandbox consumer repo by pinning to `@<sha>`.
3. **Clean input/output names**: Use `kebab-case`; descriptions in English.
4. **README per action**: Sections for `## Inputs`, `## Outputs`, `## Usage`, and `## Known limitations`.
5. **Update this file's checklist** and the [`actions/README.md`](https://github.com/tcwlab/actions/blob/main/README.md).
6. **Major tag discipline**: On breaking changes, create a new major version (`v2`); keep the old tag available for consumers.

---

## What explicitly does NOT go in this repo

- **JavaScript/TypeScript actions.** We build composite actions only — no Node runtime layer in the action source code.
- **Docker actions.** If an action needs a specific tool, that tool comes from a `tcwlab/<image>` container, which the action uses via `container:` in a step — the action itself is YAML-only.
- **Repo-specific logic.** A composite action belongs here only after **at least three consumer repos** would use the same pattern. Before that: keep it as copy-paste in consumer repos.
- **Forgejo marketplace metadata.** Forgejo has no marketplace like GitHub, so no branding fields.
- **Secrets handling in action code.** Secrets are passed from the consumer workflow as inputs — never read from environment, never logged.

---

## Consumer snippets

### Use pattern (once `trivy-scan-pr-comment` exists)

```yaml
security:
  runs-on: ubuntu-22.04
  needs: build
  steps:
    - uses: https://data.forgejo.org/actions/checkout@v4
    - uses: https://<your-forgejo-host>/tcwlab/actions/.forgejo/actions/trivy-scan-pr-comment@v1
      with:
        image: tcwlab/myservice:${{ github.sha }}
        severity: HIGH,CRITICAL
        forgejo-token: ${{ secrets.FORGEJO_TOKEN }}
```

### Action definition skeleton

`action.yml`:

```yaml
name: Trivy Scan with PR Comment
description: >
  Scans a container image with Trivy, formats the result as a Markdown table,
  and posts/updates it in the PR description.
inputs:
  image:
    description: Image ref to scan (e.g., tcwlab/myservice:abc123)
    required: true
  severity:
    description: Severity filter (comma-separated)
    required: false
    default: HIGH,CRITICAL
  forgejo-token:
    description: Token for Forgejo API (PR description update)
    required: true
runs:
  using: composite
  steps:
    - shell: bash
      run: |
        # ... Trivy run, Markdown generation, PR description update ...
```

---

## Known pain points / open questions

- **Forgejo composite action path**: Currently the `uses:` path pattern (`<base>/.forgejo/actions/<name>@<ref>`) is Forgejo-specific. GitHub Actions uses a different path. As long as we only run Forgejo, this is fine — on migration to GitHub, the path would need adjustment.
- **Action testing**: There is no native Forgejo test infrastructure for composite actions. The repo's own `ci.yml` only lints; action functionality is validated empirically in consumer repos. Idea: a sandbox test repo that smoke-tests the important actions in a regular workflow.
- **Version drift with `templates/`**: When a composite action from `actions/` is embedded in a template, the template's pin must be kept in sync. A Renovate/CI-assisted drift check is missing.
- **Minimal repo content right now**: The repo is started but empty. Next concrete step: extract `trivy-scan-pr-comment` logic from image repos.
- **Bootstrap chicken-and-egg**: The repo does not use itself (to avoid a chicken-and-egg problem), only `tcwlab/betterlint` + `tcwlab/semantic-release` from sibling repos.
