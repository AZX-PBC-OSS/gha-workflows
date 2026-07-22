# gha-workflows

Centralized, SHA-pinnable GitHub Actions **reusable workflows** for trusted
publishing (OIDC) PyPI releases.

This repository exposes a single reusable workflow,
[`publish-python.yml`](.github/workflows/publish-python.yml), that any
AZX-PBC-OSS (or third-party) repository can call to build a Python distribution,
verify the git tag matches the project version, smoke-test the wheel, and
publish to PyPI using **OIDC trusted publishing** — no stored API tokens
required.

> **Why a centralized reusable workflow?** Every caller pins the workflow to a
> specific commit SHA. A single, reviewed, audited workflow file becomes the
> source of truth for how AZX-PBC-OSS projects release to PyPI, while each
> caller retains its own trigger logic and its own PyPI trusted-publisher
> configuration.

## The reusable workflow: `publish-python.yml`

**File:** `.github/workflows/publish-python.yml`
**Trigger:** `workflow_call` only (reusable). It has **no** `push`, `release`,
or `workflow_dispatch` triggers by design — triggering is the caller's job.

### Inputs

| Input | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `dist-name` | `string` | **yes** | — | Distribution name on PyPI (the `[project] name` in `pyproject.toml`). Used for the version check and the PyPI project URL. |
| `python-version` | `string` | **yes** | — | Python version to build and test under (e.g. `"3.12"`). |
| `tag-name` | `string` | **yes** | — | Git tag to build and publish (e.g. `v0.2.0`). The workflow checks out this ref and verifies the tag (sans leading `v`) equals the installed project version. |
| `environment-name` | `string` | no | `pypi` | GitHub environment for the publish job. Must match the environment configured on the PyPI trusted publisher. |

### What it does

1. **Build job** — checks out `tag-name`, installs `uv`, sets up Python, verifies
   the git tag matches the project's declared version, builds sdist + wheel
   with `uv build`, smoke-tests the wheel by importing it in a throwaway venv,
   and uploads `dist/` as an artifact.
2. **Publish job** — downloads the artifact, runs in the configured GitHub
   environment with `id-token: write`, and publishes to PyPI via
   `pypa/gh-action-pypi-publish@release/v1` using OIDC trusted publishing.

## Caller requirements

To use this workflow, a caller repository must do **all** of the following.

### 1. Pin `uses:` to a commit SHA — never `@main`

Because invoking a reusable workflow crosses a **repository trust boundary**, a
caller must pin the `uses:` reference to a specific commit SHA of
`AZX-PBC-OSS/gha-workflows`. Do **not** use `@main` or a moving tag — a
compromise or accidental merge on `main` would silently change the release
behavior of every un-pinned caller.

```yaml
# GOOD
uses: AZX-PBC-OSS/gha-workflows/.github/workflows/publish-python.yml@<pin-sha>

# BAD — mutable ref, never use
uses: AZX-PBC-OSS/gha-workflows/.github/workflows/publish-python.yml@main
```

The SHA to pin is the `HEAD` commit on `main` of this repository. Dependabot
(see [Security](#security)) keeps that pin current.

### 2. Configure a PyPI trusted publisher

On **each** PyPI project that uses this workflow, configure an OIDC trusted
publisher with exactly these values:

| PyPI field | Value |
| --- | --- |
| PyPI project | the project being released |
| Workflow name | `publish-python.yml` |
| Repository | `AZX-PBC-OSS/gha-workflows` |
| Environment name | `pypi` (or whatever you pass as `environment-name`) |

> **Important:** the trusted publisher points at **this** repository
> (`AZX-PBC-OSS/gha-workflows`), not the caller repository. PyPI matches the
> OIDC token's `job_workflow_ref` to this workflow file, which is why the
> centralized model works without stored tokens. The `environment-name` you
> pass to the workflow must match the environment named in the PyPI trusted
> publisher, and the GitHub environment with that name must exist on
> `AZX-PBC-OSS/gha-workflows`.

### 3. Keep a per-repo `publish.yml` shim

This workflow is reusable-only, so each caller keeps its own trigger shim — a
per-repo `publish.yml` that fires on a manual tag push / `workflow_dispatch`
(e.g. for republishing) and calls this workflow. The caller shim owns the
*when*; this workflow owns the *how*.

### 4. release-please `publish` job

When using [release-please], its `publish` output job should call this workflow
directly, passing `dist-name`, `python-version`, and `tag-name`. You do **not**
need to pass `attestations: false` — it is hardcoded `false` inside this
workflow (see [Security](#security) for why). The `environment-name` input is
optional and defaults to `pypi`.

## Minimal caller example

The example below shows the two files a caller repo needs:

1. `release-please.yml` — the release-please action opens the release PR; its
   `publish` job calls this workflow after the release tag is created.
2. `publish.yml` — the per-repo shim for manual republish via `workflow_dispatch`.

Both pin this workflow to a placeholder SHA `<pin-sha>`.

### `.github/workflows/release-please.yml` (caller)

```yaml
name: release-please

on:
  push:
    branches: [main]

permissions:
  contents: write
  pull-requests: write

jobs:
  release-please:
    runs-on: ubuntu-latest
    outputs:
      released: ${{ steps.release.outputs.release_created }}
      tag-name: ${{ steps.release.outputs.tag_name }}
    steps:
      - id: release
        uses: googleapis/release-please-action@v4
        with:
          release-type: python

  publish:
    needs: [release-please]
    if: ${{ needs.release-please.outputs.released == 'true' }}
    uses: AZX-PBC-OSS/gha-workflows/.github/workflows/publish-python.yml@<pin-sha>
    with:
      dist-name: my-project
      python-version: "3.12"
      tag-name: ${{ needs.release-please.outputs.tag-name }}
```

### `.github/workflows/publish.yml` (caller shim for manual republish)

```yaml
name: publish (manual)

on:
  workflow_dispatch:
    inputs:
      tag-name:
        description: Git tag to publish (e.g. v0.2.0)
        required: true
        type: string

permissions:
  contents: read

jobs:
  publish:
    uses: AZX-PBC-OSS/gha-workflows/.github/workflows/publish-python.yml@<pin-sha>
    with:
      dist-name: my-project
      python-version: "3.12"
      tag-name: ${{ github.event.inputs.tag-name }}
```

## Security

### SHA-pinning mandate (cross-repo trust boundary)

A reusable `uses:` reference executes workflow code from **another repository**.
This is a trust boundary: a malicious or accidental commit on `main` of
`gha-workflows` would change the release behavior of every caller that follows
a mutable ref. To eliminate that risk:

* Callers **MUST** pin `uses:` to a specific commit SHA of
  `AZX-PBC-OSS/gha-workflows`. Mutable refs (`@main`, `@v1`-style tags on this
  repo) are prohibited for callers.
* The pin target is the `HEAD` commit SHA on `main` of this repository.

### OIDC trusted publishing (no stored tokens)

Publishing uses PyPI's [trusted publishing (OIDC)] — no PyPI API tokens are
stored as GitHub secrets. The publish job requests a short-lived OIDC token
from GitHub (`id-token: write`) that PyPI validates against the trusted
publisher configured in step 2 above. There is nothing to leak or rotate.

### Least-privilege permissions

* The top-level `permissions:` is `contents: read` — the minimum needed to
  check out the tag.
* Only the `publish` job escalates to `id-token: write`, and only within the
  `pypi` GitHub environment, which gates who can publish.
* No other `permissions:` scopes are requested.

### Why attestations are disabled

The workflow hardcodes `attestations: false` in the publish step. This is
intentional and **not** configurable.

PyPI's publish attestations use Sigstore to record provenance. When a workflow
is invoked as a **reusable workflow**, the Sigstore certificate records the
**caller's** top-level workflow as the source — not this repository's workflow.
PyPI's attestation verification cannot match that caller-specific certificate
against the trusted publisher (which points at `AZX-PBC-OSS/gha-workflows`), so
attestations would fail verification. The OIDC *publishing* token is unaffected:
PyPI matches the `job_workflow_ref` claim to this workflow file regardless.

See the PyPI trusted-publishing documentation for the reusable-workflow
limitation. Hardcoding `false` (rather than exposing it as an input) prevents a
caller from accidentally enabling an option that will always fail.

### Dependabot keeps pins current

* `.github/dependabot.yaml` configures the `github-actions` ecosystem with a
  weekly schedule. Dependabot opens PRs to bump the **internal** pinned action
  tags used inside `publish-python.yml` (e.g. `actions/checkout@v7`,
  `astral-sh/setup-uv@v8.3.2`, `actions/upload-artifact@v7.0.1`,
  `actions/download-artifact@v8.0.1`).
* In caller repositories, dependabot (github-actions ecosystem) bumps the
  **reusable-workflow SHA pin** (`@<pin-sha>`) to the latest `main` HEAD of this
  repo after each merge here. Both the internal action tags and the
  cross-repo workflow pin are therefore tracked and reviewable.

[trusted publishing (OIDC)]: https://docs.pypi.org/trusted-publishers/
[release-please]: https://github.com/googleapis/release-please
