# checkout-shell

Shell-based git checkout for jobs running in containers without Node.js.

## Why

`actions/checkout@v4` is a JavaScript action — it needs `node` in the
container. The tcwlab images (Alpine-based, hardened, minimal) deliberately
omit Node.js, so the JS-action fails with:

```text
exec: "node": executable file not found in $PATH
```

`checkout-shell` does the same job using `git init`, `git fetch` and
`git checkout` directly.

## Inputs

| name           | required | default                | description                                          |
|----------------|----------|------------------------|------------------------------------------------------|
| `token`        | no       | `${{ github.token }}`  | Forgejo PAT with repo:read scope                     |
| `ref`          | no       | `${{ github.sha }}`    | Git ref to check out (branch, tag, or commit SHA)    |
| `fetch-depth`  | no       | `1`                    | Number of commits to fetch (`0` = full history)      |
| `forgejo-host` | no       | `git.mon.k8b.co`       | Forgejo host (no protocol)                           |

## Usage

### Minimal (shallow checkout of the current commit)

```yaml
jobs:
  lint:
    runs-on: ubuntu-22.04
    container:
      image: tcwlab/betterlint:2.17.1
    steps:
      - uses: https://github.com/tcwlab/actions/.forgejo/actions/checkout-shell@v1
      - run: betterlint --dir .
```

### Full history (e.g. for semantic-release)

```yaml
- uses: https://github.com/tcwlab/actions/.forgejo/actions/checkout-shell@v1
  with:
    fetch-depth: "0"
```

### Custom token (e.g. cross-repo workflows)

```yaml
- uses: https://github.com/tcwlab/actions/.forgejo/actions/checkout-shell@v1
  with:
    token: ${{ secrets.FORGEJO_BOT_TOKEN }}
```

## Notes

- The action sets `safe.directory` for the current `$PWD` to silence
  newer git's ownership-mismatch error in container-mounted workspaces.
- `fetch-depth: 1` is the default for speed. Set `0` if your subsequent
  steps need full tags or history (semantic-release, git-describe, etc.).
- The action targets Forgejo URL shapes (`/owner/repo.git`). For
  cross-host workflows, override `forgejo-host`.
