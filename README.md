# EffortlessMetrics public CI workflow surface

GitHub does not allow a public repository to call a reusable workflow stored in
an internal repository. This repository is the public, workflow-only distribution
of internal `EffortlessMetrics/em-ci`.

It contains no host inventory, credentials, runner-management code, provider
addresses, or private configuration. Public and internal consumers pin the
reusable workflow by full commit SHA:

```yaml
jobs:
  ci:
    uses: EffortlessMetrics/em-ci-workflows/.github/workflows/rust.yml@FULL_COMMIT_SHA
    with:
      profile: standard
      script: .ci/run.sh
      fetch_depth: 1
```

Use `fetch_depth: 0` only when the proof script compares revisions, such as
`cargo-allow diff --base origin/$GITHUB_BASE_REF`.

Profiles are capability contracts:

- `light`: 3 CPU / 6 GiB
- `standard`: 4 CPU / 9 GiB
- `heavy`: 6 CPU / 12–13 GiB
- `large`: exclusive 14 CPU / 28 GiB CX53 mode

The workflow's small `ci-control` jobs execute only this full-SHA-pinned workflow
source. They do not check out caller-controlled source or receive repository
secrets. Actual Rust proof runs only on the requested capability.

The workflow routes same-repository, non-bot PRs and approved owned events to the
trusted self-hosted pool. Fork and bot-authored PRs run on GitHub-hosted Ubuntu
using the same public runner image only when:

```text
EM_CI_GITHUB_HOSTED_ENABLED=true
```

While hosted execution is disabled, external source is never checked out on the
trusted fleet. An explicit unavailable implementation makes the normalized
result red. Unknown event types fail closed.

Consumers configure organization or repository variables:

```text
EM_CI_RUNNER_IMAGE=ghcr.io/effortlessmetrics/em-ci-rust-runner@sha256:...
EM_CI_GITHUB_HOSTED_ENABLED=false|true
```

`EM_CI_RUNNER_IMAGE` must be an immutable GHCR digest. The GHCR package must
allow anonymous pulls before hosted external-PR jobs are enabled.

Branch protection should require only:

```text
Rust CI / Required
```

Do not require conditional implementation jobs.

The internal source of truth remains `.github/workflows/rust.yml` in `em-ci`.
`em-ci` validation requires this public workflow and actionlint configuration to
remain byte-for-byte synchronized with that source. Changes are published from
internal `em-ci`; this repository is not edited independently.
