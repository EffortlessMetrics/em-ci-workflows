# EffortlessMetrics public CI workflow surface

GitHub does not allow a public repository to call a reusable workflow stored in
an internal repository. This repository is the generated public workflow
distribution of internal `EffortlessMetrics/em-ci`.

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

- `light`: full 4 guest vCPUs / 4 build/test threads / 6 GiB
- `standard`: full 6 guest vCPUs / 6 build/test threads / 9 GiB on VPS20
- `heavy`: CX43 full 8 guest vCPUs / 8 build/test threads / 13 GiB; CX53 normal sees all 16 guest vCPUs with 8 build/test threads / 12 GiB
- `large`: exclusive CX53 with full 16 guest vCPUs / 16 build/test threads / 28 GiB

CPU access is work-conserving. A slot's Docker ceiling equals the host's full
guest-vCPU count; no profile pins CPU IDs or withholds whole guest CPUs for host
overhead. Both CX53 normal slots use the full 16-vCPU ceiling and equal CPU
weight, so one active job can use idle capacity while two active jobs share the
machine. Memory, no-swap, disk, inodes, PIDs, build/test parallelism, and
simultaneous slot count remain independent boundaries.

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

The internal source of truth remains `EffortlessMetrics/em-ci`. The public
payload is declared and audited there, then published through a generated pull
request. This repository is not maintained independently. README, license,
security-policy, or source-marker-only changes do not change the executable
workflow contract; callers move only when the workflow blob changes.

## Security

Sensitive findings must not be opened as public issues. Follow `SECURITY.md` and
use GitHub's private vulnerability-reporting or security-advisory surface. The
public repository contains no operational credential or host inventory.

## License

The public workflow distribution is available under either of:

- Apache License, Version 2.0 (`LICENSE-APACHE`)
- MIT License (`LICENSE-MIT`)

at your option.
