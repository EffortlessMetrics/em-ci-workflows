# EffortlessMetrics public CI workflow surface

GitHub does not allow a public repository to call a reusable workflow stored in
an internal repository. This repository is the generated public workflow
distribution of internal `EffortlessMetrics/em-ci`.

It contains no host inventory, credentials, runner-management code, provider
addresses, or private configuration. Public and internal consumers pin the
reusable workflow by full commit SHA.

## Rust consumer

```yaml
permissions:
  contents: read
  checks: read
  actions: read
  pull-requests: read
jobs:
  ci:
    uses: EffortlessMetrics/em-ci-workflows/.github/workflows/rust.yml@FULL_COMMIT_SHA
    with:
      profile: standard
      script: .ci/run.sh
      fetch_depth: 1
```

Rust control/result jobs use the owned `rust-standard` pool on the normal path.
GitHub performs native queueing; no hosted observer waits for owned capacity.
An explicit enabled hosted selection and the existing typed pre-proof recovery
can select hosted execution. Queueing alone does not promote work to hosted.

An optional `result_script` runs in a separate repository-policy job after exact
proof validation, from a credential-free checkout of the same event SHA. It receives
`EM_CI_EVIDENCE` and `EM_CI_SELECTED_PROOF_RESULT`; its failure fails the reusable
call. It has the read scopes above, never a write token or inherited secrets.
Callers must require the complete reusable call to succeed, not only inspect the
selected-proof envelope. The immutable final job independently validates proof,
requires the configured policy job to succeed, publishes its own envelope and exposes
its collision-resistant artifact name as `receipt_name`; no separate receipt
runner is needed. Any required-check migration must retain the complete
repository result policy and be qualified before changing the binding.

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

Rust classification, refusal and final result jobs execute only immutable workflow
code, with no caller checkout. A separate ordinary execution job optionally runs
the chosen repository result policy after exact proof validation, with read-only
access and a 35-minute limit. The final aggregate uses the native policy-job result,
never caller-produced evidence. The optional policy adds one owned job on the normal
path; final control and receipt stay immutable on owned standard capacity. Python's
existing pure control jobs still use GitHub-hosted Ubuntu and do not check out
caller-controlled source or receive repository secrets.

Trusted proofs normally run on the requested self-hosted language capability.
When hosted execution is enabled, Rust callers may request `force_hosted: true`.
The Rust workflow may also recover on GitHub-hosted Ubuntu after an explicit
pre-proof `toolchain_unavailable` or `disposable_paths_unavailable` admission
withdrawal. These hosted routes use the same requested profile and immutable
runner image. Compiler, test, policy, cancellation and ambiguous failures stay
blocking. Missing runners may remain queued; this contract does not provide
prompt automatic recovery from an unmatched self-hosted queue.

The default route for same-repository, non-bot PRs and approved owned events is
the trusted self-hosted pool, subject to the enabled Rust hosted routes above.
Fork and bot-authored PRs run on GitHub-hosted
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

## Optional canonical RIPR receipts

Rust callers may set `upload_artifacts: true`, `artifact_lane: ripr` and a
repository `result_script`. The proof exports current canonical receipts under
the fresh fixed `.ci/artifacts/ripr` root exposed as `EM_CI_ARTIFACT_DIR`.
Packages include the reserved finalized `em-ci-job-context.v1`, with a maximum
of 512 regular files and 8GiB including context. Required upload failure makes
the producer unsuccessful. Failed-proof diagnostics cannot create proof.

The separate repository-policy job authenticates the current native producer
attachment, streams and verifies its digest, safely extracts bounded complete
receipts, and exposes only that verified directory to policy. The defining
workflow must be pinned by full SHA; caller and native PR/evaluated merge
identities remain distinct. Policy retains repository domain validation and
receives `EM_CI_SELECTED_PROOF_RESULT` while the call is unfinished. External
consumers must still require actual completed-call success.

The immutable aggregate executes no caller code or archive import and withholds
successful outputs until native policy success and its own small upload pass.
Default callers are unchanged. Real native/domain/resource qualification and
the exact workflow allowance precede migration or required-check cutover.

## Security

Sensitive findings must not be opened as public issues. Follow `SECURITY.md` and
use GitHub's private vulnerability-reporting or security-advisory surface. The
public repository contains no operational credential or host inventory.

## License

The public workflow distribution is available under either of:

- Apache License, Version 2.0 (`LICENSE-APACHE`)
- MIT License (`LICENSE-MIT`)

at your option.
