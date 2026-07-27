# Security

This repository is a generated executable distribution of internal
`EffortlessMetrics/em-ci`. Do not submit independent workflow changes here.

Do not open a public issue for a finding that could:

- expose runner-manager, publisher, review, or release credentials;
- route fork or bot code onto trusted self-hosted capacity;
- bypass the full-SHA workflow restriction;
- poison persistent Cargo or sccache state;
- escape the runner container or write outside the bounded CI filesystem; or
- alter image or workflow identity without the reviewed activation path.

Use GitHub's private vulnerability-reporting or security-advisory surface for
this repository. Include the affected public commit, the consuming event/trust
context, a minimal reproduction that does not target production hosts, and the
containment or credential rotation you believe is required.

Non-sensitive workflow defects should be reported in the consuming
EffortlessMetrics repository where they are observed. The fix belongs in the
internal source and reaches this repository through a generated pull request.
