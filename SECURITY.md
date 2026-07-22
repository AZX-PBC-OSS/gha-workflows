# Security Policy

## Reporting a Vulnerability

We take security vulnerabilities seriously. If you discover a security issue
in this repository, **please do not open a public GitHub issue**.

Instead, report it privately using GitHub's security advisory feature:

1. Go to **Security** > **Advisories** > **Report a vulnerability** on this
   repository, or visit
   <https://github.com/AZX-PBC-OSS/gha-workflows/security/advisories/new>.
2. Provide a clear description of the issue, steps to reproduce, and any
   relevant proof of concept.
3. We will acknowledge receipt within **5 business days** and aim to publish a
   fix and advisory within **30 days** for confirmed issues, depending on
   severity.

Alternatively, you may email security findings to the AZX, PBC open-source
mailbox: `oss@azx.io`.

## Trust Model

This repository publishes **reusable GitHub Actions workflows** that other
repositories invoke via `uses:`. Because the `uses:` reference crosses a
repository trust boundary, callers MUST pin the workflow to a specific commit
SHA rather than a mutable branch or tag. See the **Security** section of the
[README](README.md#security) for the full policy, including why attestations
are disabled and how dependabot keeps pins current.
