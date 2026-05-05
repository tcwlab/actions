# tcwlab/actions

Forgejo composite actions for the tcwlab toolchain — placeholder for reusable CI building blocks.

[![License: Apache 2.0](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](LICENSE)

---

## What is this?

Reusable YAML building blocks (Forgejo composite actions) that consumer repos can embed in their `.forgejo/workflows/ci.yml` files. Each action is defined under `.forgejo/actions/<name>/action.yml` with its own README documenting inputs, outputs, and usage examples.

---

## Status

🔲 **Under construction.** This repo is initialized but currently contains no composite actions. The first actions are planned and will land as concrete PR implementations from the tcwlab team:

| Action | Purpose | Status |
|--------|---------|--------|
| `trivy-scan-pr-comment` | Trivy image scan + Markdown report + PR description update | Planned |
| `semantic-release-forgejo` | semantic-release wrapper with Forgejo variables + `.NEXT_RELEASE_VERSION` output | Planned |
| `docker-buildx-publish` | Multi-architecture build + Docker Hub push with auto-tagging | Planned |
| `betterlint-with-pr-comment` | Lint run + optional PR comment on findings | Planned |

Once implemented, each action gets its own subfolder with `action.yml` plus a detailed README.

---

## Usage pattern (once actions exist)

```yaml
steps:
  - uses: https://data.forgejo.org/actions/checkout@v4

  - name: Trivy Scan with PR Comment
    uses: https://<your-forgejo-host>/tcwlab/actions/.forgejo/actions/trivy-scan-pr-comment@v1
    with:
      image: tcwlab/myservice:${{ github.sha }}
      severity: HIGH,CRITICAL
      forgejo-token: ${{ secrets.FORGEJO_TOKEN }}
```

> **Note**: Replace `<your-forgejo-host>` with the Forgejo instance that hosts a mirror of this repo (Forgejo composite actions are looked up by URL at runtime, so the host must be reachable from your Forgejo runners).

---

## Version pinning

Composite actions are versioned via **Git tags in this repo** (no Docker Hub artifact). Consumers pin using the Git ref:

> Version numbers below are illustrative. For the current set of tags, see
> [GitHub releases](https://github.com/tcwlab/actions/releases).

| Pin format | Use case |
|----------|----------|
| `@v1` | Major tag; tracks patches and minor releases. Default for internal consumers. |
| `@v1.2.0` | Concrete SemVer. Recommended for production and audit-required repos. |
| `@<sha>` | Immutable commit hash for strict reproducibility and compliance. |

This repo uses semantic-release to automatically tag releases from Conventional Commits.

---

## How to add a new composite action

1. **Open a PR** on a `claude/feat-<action-name>` branch with a new subfolder `.forgejo/actions/<name>/` containing:
   - `action.yml` — Forgejo composite action definition with `name`, `description`, `inputs:`, `outputs:`, `runs.using: composite`, and `runs.steps: [...]`
   - `README.md` — documentation with sections for **Inputs**, **Outputs**, and **Usage example**

2. **Test locally** in a sandbox consumer repo by pinning to `@<sha>` before release.

3. **Use kebab-case** for input/output names. Descriptions should be clear and English-first.

4. **Update this file's status table** and link your action in the README.

5. **Follow the release discipline**: semantic-release automatically creates Git tags; major version tags (`v1`) move on compatible releases. When breaking changes occur, bump the major version (`v2`) and keep old tags available for existing consumers.

---

## What does NOT go here

- **JavaScript/TypeScript actions** — we build composite actions only (YAML-based). No Node runtime layer.
- **Docker actions** — if an action needs a specific tool, that tool comes from a `tcwlab/<image>` container referenced in the action's steps, not baked into the action itself.
- **Single-repo logic** — composite actions are shared only after **at least three consumer repos** use the same pattern. Before that, copy-paste the logic into each consumer repo.
- **Secrets in action code** — secrets are passed from consumer workflows as inputs, never read from environment or logged.

---

## Source

- **Source**: [github.com/tcwlab/actions](https://github.com/tcwlab/actions)
- **Issues**: [github.com/tcwlab/actions/issues](https://github.com/tcwlab/actions/issues)

---

## License

Apache-2.0 — The Chameleon Way.
