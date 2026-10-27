# Translatize Sync – GitHub Action

**Translation management with git-like branching.** This action wraps the
[`@translatize/cli`](https://www.npmjs.com/package/@translatize/cli) so you can
keep your repository and your [Translatize](https://app.translatize.com) project
in sync from CI – push new source keys, pull completed translations, and gate
pull requests on missing or changed strings.

---

## What it does

The action runs a single Translatize CLI command inside your workflow:

| Command  | What it does |
| -------- | ------------ |
| `push`   | Uploads source keys from your repo to the token's Translatize branch. |
| `pull`   | Downloads translations from the token's branch into your local files. |
| `status` | Reports the diff between repo and branch; can fail the job on missing or changed strings. |

It authenticates with a **branch-bound API token** (`mcni_...`), which the action
passes to the CLI as the `TRANSLATIZE_API_TOKEN` environment variable. A token
carries a base branch, so most workflows never set a branch at all. A token whose
scope is `create-own` may additionally target branches **it created** – pass
`--branch <name>` through `args` (or set `branch` in the config) to sync one.

Under the hood the step runs:

```
npx --yes @translatize/cli@<version> <command> --config <config> [--api-url <api-url>] <args>
```

`status` follows the CLI exit-code contract – `0` on success; `1` when a
`--fail-on-*` gate trips **or** an API/execution error occurs (e.g. an expired
token); `2` on a usage or configuration error (bad flags, missing/invalid config,
no token) – so a failed check fails the job.

---

## Quickstart

1. **Create a token.** In Translatize, open **Project Settings -> Integrations**
   and create an API token. Pick the branch it should be bound to (e.g. `main`).
   Copy the `mcni_...` value – it is shown only once.
2. **Store it as a secret.** In your GitHub repo, go to
   **Settings -> Secrets and variables -> Actions -> New repository secret**,
   name it `TRANSLATIZE_TOKEN`, and paste the token.
3. **Commit your config.** Add a `translatize.config.json` at the repo root – the
   same file the `translatize` CLI reads locally (project id, branch, and the
   file paths it manages). See the CLI documentation for its schema.
4. **Add a workflow.** Start with the status gate below, then add push/pull.

Minimal example:

```yaml
name: Translatize
on: [workflow_dispatch]
jobs:
  sync:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: translatize/sync-action@v1
        with:
          command: status
          token: ${{ secrets.TRANSLATIZE_TOKEN }}
```

---

## Workflow recipes

### (a) Push new source keys on every push to `main`

Whenever your source strings change on `main`, upload the new keys to Translatize
so translators can start working.

```yaml
name: Push source keys to Translatize

on:
  push:
    branches: [main]
    paths:
      - 'locales/en.json'
      - 'src/**/*.json'

jobs:
  push:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Push source keys
        uses: translatize/sync-action@v1
        with:
          command: push
          token: ${{ secrets.TRANSLATIZE_TOKEN }}
```

### (b) Nightly translation pull that opens a pull request

Pull finished translations every night and open a PR with the changes, so a human
reviews and merges them.

```yaml
name: Nightly translation pull

on:
  schedule:
    - cron: '0 3 * * *'
  workflow_dispatch:

permissions:
  contents: write
  pull-requests: write

jobs:
  pull:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Pull latest translations
        uses: translatize/sync-action@v1
        with:
          command: pull
          token: ${{ secrets.TRANSLATIZE_TOKEN }}

      - name: Open a pull request
        uses: peter-evans/create-pull-request@v7
        with:
          commit-message: 'chore(i18n): sync translations from Translatize'
          title: 'Update translations from Translatize'
          body: 'Automated nightly translation sync from the Translatize branch bound to TRANSLATIZE_TOKEN.'
          branch: chore/translatize-sync
          labels: i18n, automated
```

> `create-pull-request` needs the `contents: write` and `pull-requests: write`
> permissions shown above. If the PR step fails to open a PR, also enable
> **Settings -> Actions -> General -> Allow GitHub Actions to create and approve
> pull requests**.

### (c) Block a PR when translations are missing

Run `status --fail-on-missing` on pull requests so a PR that adds untranslated
source keys turns the check red until translations land.

```yaml
name: Translation status gate

on:
  pull_request:
    branches: [main]

jobs:
  status:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Fail if translations are missing
        uses: translatize/sync-action@v1
        with:
          command: status
          token: ${{ secrets.TRANSLATIZE_TOKEN }}
          args: --fail-on-missing
```

> Swap `--fail-on-missing` for `--fail-on-diff` to instead fail when a source
> string changed but its translations were not updated. Pass both to enforce
> either condition.

---

## Inputs

| Input          | Required | Default                    | Description |
| -------------- | -------- | -------------------------- | ----------- |
| `command`      | yes      | –                          | CLI command to run: `pull`, `push`, or `status`. Any other value fails the step. |
| `token`        | yes      | –                          | Branch-bound Translatize API token (`mcni_...`). Passed to the CLI as `TRANSLATIZE_API_TOKEN`. Use a secret. |
| `version`      | no       | `latest`                   | `@translatize/cli` npm version to run (e.g. `1.4.0`). |
| `config`       | no       | `translatize.config.json`  | Path to the CLI config file, relative to the workspace. |
| `api-url`      | no       | `""`                       | Override the Translatize API base URL (self-hosted / staging). Empty uses the CLI default. |
| `args`         | no       | `""`                       | Extra CLI flags appended verbatim, e.g. `--dry-run`, `--json`, `--fail-on-missing`, `--fail-on-diff`. |
| `node-version` | no       | `24`                       | Node.js version used to run the CLI. |

This action has no outputs – surface results through the step's exit status
(a red check) and the CLI's `--json` output in the logs.

---

## Setting up the token secret

- Tokens are created in **Project Settings -> Integrations** at
  [app.translatize.com](https://app.translatize.com) and are prefixed `mcni_`.
- Store the value as a repository (or environment/organization) secret named
  **`TRANSLATIZE_TOKEN`** and reference it as
  `token: ${{ secrets.TRANSLATIZE_TOKEN }}`. Never paste the raw token into a
  workflow file.
- A token carries its own **role** (translator can write translations;
  developer/admin/owner can also create source keys) and may be **IP-allowlisted**
  or given an **expiry** – set these to match what the workflow needs.

### One token per branch

**Each API token has a base branch, chosen when you mint it.** A standard
(`fixed`-scope) token may act only on that branch, so naming a different one is
rejected by the API – the branch is decided by the token, not the workflow. (A
`create-own`-scope token is the exception: it may also target branches it created,
selected with `--branch` via `args` or the config `branch` field.) Teams that sync
several branches with fixed-scope tokens typically create **one token per branch**:

- a `main`-bound token used by the push/pull/status workflows on `main`;
- a separate token bound to a `staging` (or feature) branch, stored as a second
  secret, for workflows that sync that branch.

Rotate a token by creating a new one and updating the secret; revoking the old
token in Translatize immediately invalidates it.

---

## Security notes

- Every workflow input is forwarded to the CLI through environment variables and
  referenced as quoted shell variables – no `${{ inputs.* }}` value is ever
  interpolated into the shell script, which closes the usual script-injection
  hole in composite actions.
- The `command` input is validated against `pull | push | status`; anything else
  fails the step before the CLI runs.
- Pin third-party actions to a tag or, for stronger supply-chain guarantees, to a
  full commit SHA (this action, `actions/checkout`, `actions/setup-node`, and
  `peter-evans/create-pull-request`).

---

## Publishing note

This action is published to the
**[GitHub Marketplace](https://github.com/marketplace/actions/translatize-sync)**;
its source lives at
**[github.com/Translatize/sync-action](https://github.com/Translatize/sync-action)**.
Reference it exactly as the examples above do:

```yaml
uses: translatize/sync-action@v1
```

### Alternatives

You can also consume the action without the Marketplace – both take the same
inputs as the published version:

- **Local path** – copy this `github-action/` folder into your repository (for
  example to `.github/actions/translatize`) and reference it by path:

  ```yaml
  uses: ./.github/actions/translatize
  ```

- **Fork** – mirror/fork the folder into a repo you control and pin to a tag or
  commit SHA:

  ```yaml
  uses: your-org/translatize-sync-action@<sha>
  ```
