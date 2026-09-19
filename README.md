# EffortlessMetrics public CI workflow surface

GitHub does not allow a public repository to call a reusable workflow stored in
an internal repository. This repository is the generated public workflow
distribution of internal `EffortlessMetrics/em-ci`.

It contains no host inventory, credentials, runner-management code, provider
addresses, or private configuration. Public and internal consumers pin the
reusable workflow by full commit SHA.

## Rust consumer

```yaml
jobs:
  ci:
    uses: EffortlessMetrics/em-ci-workflows/.github/workflows/rust.yml@FULL_COMMIT_SHA
    with:
      profile: standard
      script: .ci/run.sh
      fetch_depth: 1
```

## Python consumer

```yaml
jobs:
  python:
    uses: EffortlessMetrics/em-ci-workflows/.github/workflows/python.yml@FULL_COMMIT_SHA
    with:
      profile: standard
      python: '3.13'
      lane: verify
      script: .ci/python-verify.sh
      fetch_depth: 1
      upload_artifacts: true
      require_media: false
```

The Python proof script receives:

```text
EM_CI_PYTHON
EM_CI_PIP
EM_CI_PYTHON_SERIES
EM_CI_PYTHON_VERSION
EM_CI_LANE
EM_CI_REQUIRE_MEDIA
EM_CI_TARGET_REPOSITORY
EM_CI_TARGET_SHA
EM_CI_ARTIFACT_DIR
```

`EM_CI_PYTHON` and `EM_CI_PIP` point into a fresh per-job venv created from the
exact pinned Python runtime. A proof may write bounded retained evidence only
under `.ci/artifacts/<lane>`; the central workflow owns artifact publication.
The lane must be a lowercase safe identifier and is used in the artifact name.
`require_media: true` adds a fixed SHA-256-verified FFmpeg/ffprobe archive to the selected
self-hosted Python proof path, records the two safe version lines, and exports
`EM_CI_REQUIRE_MEDIA=true` to the repository proof. It does not change trust classification;
hosted external jobs receive the requested value but do not install the self-hosted overlay.

The classifier derives the source repository and immutable commit from the
triggering event. Pull requests use `pull_request.head.repo.full_name` and
`pull_request.head.sha`, not GitHub's synthetic merge ref. Owned push,
merge-group, scheduled, and manually dispatched events use their exact event
repository and SHA. Checkout receives both classified values explicitly and the
workflow compares `git rev-parse HEAD` with the classified SHA before any
repository script executes. Job context retains the target SHA separately from
the event SHA.

Use `fetch_depth: 0` only when the proof script compares revisions. Consumer
workflows cannot provide `runs-on`, a container image, an arbitrary shell body,
or an artifact path.

Profiles are capability contracts:

- `light`: full 4 guest vCPUs / 4 build/test threads / 6 GiB
- `standard`: full 6 guest vCPUs / 6 build/test threads / up to 10240 MiB on VPS20
- `heavy`: CX43 full 8 guest vCPUs / 8 build/test threads / 13 GiB; CX53 normal sees all 16 guest vCPUs with 8 build/test threads / 12 GiB
- `large`: exclusive CX53 with full 16 guest vCPUs / 16 build/test threads / 28 GiB

Rust and Python use parallel language-specific labels over those envelopes. The
consumer selects only the reviewed profile input; the reusable workflow maps it
to one fixed capability label.

CPU and memory access are work-conserving. On a single-slot VPS20, the 9984-MiB soft boundary and 10240-MiB hard cgroup ceiling contain pressure/runaway use without treating the hard ceiling as pre-allocated RAM. CPU access is work-conserving. A slot's Docker ceiling equals the host's full
guest-vCPU count; no profile pins CPU IDs or withholds whole guest CPUs for host
overhead. Both CX53 normal slots use the full 16-vCPU ceiling and equal CPU
weight, so one active job can use idle capacity while two active jobs share the
machine. Memory, no-swap, disk, inodes, PIDs, language-specific parallelism, and
simultaneous slot count remain independent boundaries.

The workflows' pure control jobs (`classify`, `external-unavailable`, and `result`) execute
on GitHub-hosted Ubuntu using only full-SHA-pinned workflow source. They do not check
out caller-controlled source or receive repository secrets. Actual proof runs only on the
requested language capability.

The workflows route same-repository, non-bot PRs and approved owned events to
the trusted self-hosted pool. Fork and bot-authored PRs run on GitHub-hosted
Ubuntu using the same public runner image only when:

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

Branch protection should require only the normalized result for the language
workflow the repository uses:

```text
Rust CI / Required
Python CI / Required
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
