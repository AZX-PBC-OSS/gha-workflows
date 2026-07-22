# Contributing to `gha-workflows`

This repository holds centralized, **reusable** GitHub Actions workflows used by
AZX-PBC-OSS open-source projects. Changes here affect every repository that
pins to this workflow, so they require extra care.

## Proposing Changes

1. **Open an issue first** describing the proposed change, its motivation, and
   which caller repositories will be affected. Significant behavioral changes
   should be discussed before a PR is opened.
2. Fork the repository and create a feature branch from `main`.
3. Make your change to the workflow under `.github/workflows/`. Keep the
   workflow **reusable-only** (`workflow_call` trigger) — do **not** add
   `push`, `workflow_dispatch`, or `release` triggers here. Per-repo trigger
   logic belongs in each caller's own shim, not in this centralized workflow.
4. Open a pull request. Link the issue from step 1 and describe how the change
   was validated (see below).

## SHA-Pinning Policy

Callers MUST pin `uses:` to this repository's workflow at a specific commit
SHA, never to `@main` or a moving tag. Because a new commit on `main` shifts
the pin target, every merge to `main` effectively produces a new candidate pin
for callers. There is **no floating reference** that callers should follow.

When you merge a change:

* The new `main` HEAD commit SHA becomes the next pin target for callers.
* Dependabot (github-actions ecosystem) will open PRs in caller repos to bump
  their pin to the new SHA — this is configured per caller repo, not here.
* Dependabot in **this** repo tracks the pinned third-party action versions used
  inside the workflow (e.g. `actions/checkout`, `astral-sh/setup-uv`,
  `pypa/gh-action-pypi-publish`) via `.github/dependabot.yaml`.

## How to Test Changes

Reusable workflows cannot be run in isolation on a `pull_request` event — they
require a caller. Therefore, the practical validation strategy is:

1. **Workflow syntax validation.** Confirm the YAML parses and the
   `workflow_call` inputs are well-formed:
   ```bash
   uv run --with pyyaml python -c "import yaml; yaml.safe_load(open('.github/workflows/publish-python.yml'))"
   ```
2. **Action lint.** Run `actionlint` against the workflow if available:
   ```bash
   actionlint .github/workflows/publish-python.yml
   ```
3. **Test by running a real release.** The authoritative test is to point a
   caller repository at the proposed SHA (via a PR branch or a temporary tag on
   a fork) and perform an actual PyPI release using OIDC trusted publishing.
   Document the release performed (project, version, PyPI URL) in the PR
   description. A release that succeeds end-to-end (tag check → build →
   smoke-test → OIDC publish) is the strongest evidence the change is safe.

If you cannot perform a real release, state that explicitly in the PR and
request a maintainer to validate before merge.
