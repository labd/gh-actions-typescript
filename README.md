# gh-actions-typescript

Shared GitHub Actions for Lab Digital TypeScript projects.

## `pnpm-install`

Composite action that sets up Node.js and pnpm, caches the pnpm store, audits
dependencies for critical vulnerabilities and runs `pnpm install`.

Steps:

1. `actions/setup-node` (`node-version` or `node-version-file`, optional `registry-url`)
2. `pnpm/action-setup` (version from `pnpm-version` or the `packageManager` field in `<cwd>/package.json`)
3. `pnpm store path` + `actions/cache` (skipped when `cache: "false"`)
4. `pnpm audit --audit-level=critical --ignore-registry-errors`
5. `pnpm install`

### Usage

Pin the action to a commit SHA with the tag as comment (`pinact run` does this for you):

```yaml
steps:
  - uses: actions/checkout@<sha> # vX.Y.Z
    with:
      persist-credentials: false

  - uses: labd/gh-actions-typescript/pnpm-install@<sha> # v1.1.0
    with:
      node-version: "24"

  - run: pnpm test
```

Monorepo / sub directory:

```yaml
- uses: labd/gh-actions-typescript/pnpm-install@<sha> # v1.1.0
  with:
    cwd: frontend
    node-version-file: frontend/.nvmrc
```

### Release / publish workflows

Do not use the dependency cache in workflows that publish packages or create
releases: a cache entry written by another (less trusted) workflow could end up
in your release (see zizmor's
[cache-poisoning](https://docs.zizmor.sh/audits/#cache-poisoning) audit).

Set `registry-url` so `actions/setup-node` writes an `.npmrc` that authenticates
with `NODE_AUTH_TOKEN`; changesets/action v2 no longer writes one for you.

```yaml
name: Release

on:
  push:
    branches: [main]

permissions: {}

jobs:
  release:
    runs-on: ubuntu-latest
    permissions:
      contents: write
      pull-requests: write
      id-token: write
    steps:
      - uses: actions/checkout@<sha> # vX.Y.Z
        with:
          persist-credentials: false

      - uses: labd/gh-actions-typescript/pnpm-install@<sha> # v1.1.0
        with:
          cache: "false"
          registry-url: https://registry.npmjs.org

      - uses: changesets/action@<sha> # vX.Y.Z
        with:
          publish: pnpm publish-packages
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}
```

### Inputs

All inputs are optional.

| Input | Default | Description |
| --- | --- | --- |
| `node-version` | `24` | Node.js version spec (e.g. `22`, `24.x`). Ignored when `node-version-file` is set. |
| `node-version-file` | `""` | File containing the Node.js version (`.nvmrc`, `.node-version`, `.tool-versions`, `package.json`, ...), relative to the repository root. Takes precedence over `node-version`. |
| `pnpm-version` | `""` | pnpm version. When empty, the `packageManager` field of `<cwd>/package.json` is used. |
| `cwd` | `.` | Directory containing `package.json`; audit and install run here. |
| `cache` | `true` | Cache the pnpm store with `actions/cache`. Set to `false` in release/publish workflows; the store path and cache steps are skipped entirely. |
| `registry-url` | `""` | Registry URL passed to `actions/setup-node`, e.g. `https://registry.npmjs.org`. Writes an `.npmrc` that reads the token from `NODE_AUTH_TOKEN`. |
| `frozen-lockfile` | `""` | `true` runs `pnpm install --frozen-lockfile`, `false` runs `pnpm install --no-frozen-lockfile`. Empty runs plain `pnpm install`, which pnpm treats as frozen when `CI=true` (always the case on GitHub-hosted runners). |
