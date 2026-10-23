# cortex

## Purpose

cortex is the central planning and architectural decision repository for the M1 milestone build of the 3AM platform. It contains the step-by-step implementation plans (2026-10-18-m1-step0-foundation.md through 2026-10-18-m1-step13-shadow-trust-corpus.md) that guide the construction of four private Go repos under github.com/3am-si: 3am-proto, 3am-control, 3am-cli, and 3am-deploy. The plans follow test-driven development (TDD) form: each step begins with a failing test, implements code to pass it, and ends with an acceptance script whose output is pinned as evidence. Cortex does not serve user traffic; it governs how the four repos are built, tested, and released.

## Plan Task It Implements

cortex hosts the M1 Step 0 foundation plan and all subsequent steps (1–13). Step 0 defines the core architecture: a signed control-plane image serving a single ConnectRPC GetVersion call, and a 3am CLI that calls it with GKE Workload Identity in GCP and EKS Pod Identity in AWS, with no static keys anywhere. The plan tasks build the four repos in order: Task 1 creates 3am-proto's buf schema and codegen for Go/Rust/TS/Python; Task 2 builds 3am-control's auth interceptor and Google verifier; Task 3 adds the AWS STS GetCallerIdentity verifier; Task 4 implements the VersionService and controlplane server binary; Task 5 adds the 3am CLI `3am version` command; Task 6 creates the GCP lab Terraform skeleton; Task 7 creates the AWS lab Terraform skeleton; Task 8 creates signed release workflows for 3am-control and 3am-cli using ko and cosign with SPDX SBOM attestations; Task 9 adds a Helm chart and step-0 acceptance run. Each step's acceptance criteria are verbatim from the PLAN.md spec and must pass before the step is considered complete.

## Interfaces and Resources

- **PLAN.md** (root): The spec containing 14 steps, founder decisions, repositories, invariants, and open questions. The "Milestone 1 → Build order" table maps each step to its acceptance clause.
- **docs/superpowers/plans/**: Contains the 14 step plans (step0 through step13). Each plan file follows the same structure: Goal, Architecture, Global Constraints, Decisions made in this plan, Review Focus, Before you start, Task descriptions with checkbox steps, and Task 9's helm/acceptance content.
- **HOW-THESE-PLANS-ARE-MADE.md**: A checklist used to write the M1 step plans. It records the reasoning so a future writer can make the same calls or change them on purpose. Ground rules include: PLAN.md is the spec; a step plan never contradicts it silently; write one plan per build step; plans follow TDD form (failing test → run → implement → run → commit); write in plain full sentences (caveman-style compression applies to chat only, never to files).
- **Toolchain**: Go ≥ 1.26, buf, Terraform ≥ 1.9, ko, cosign, Helm 3, GitHub Actions. Scripts run under bash 3.2 with no GNU-only flags.
- **Repos governed**: 3am-proto (API schema and generated code), 3am-control (ConnectRPC server with auth), 3am-cli (3am binary), 3am-deploy (Terraform skeletons for GKE and EKS labs, registries, KMS signing key, GitHub OIDC trust, Helm chart, acceptance script).
- **Identity and trust**: No static keys. Identity comes only from GKE Workload Identity Federation, EKS Pod Identity, and GitHub OIDC. The control plane allowlist is fail-closed (empty THREEAM_ALLOWED_PRINCIPALS stops the server). Cosign signing uses a GCP KMS key with `--tlog-upload=false`; keyless signing is avoided because it would publish private repo and workflow names to the public Rekor log. Verification uses `--insecure-ignore-tlog=true` with the KMS public key.
- **API**: ConnectRPC. The proto package is `threeam.meta.v1`. The single RPC is `GetVersion(GetVersionRequest) returns (GetVersionResponse)`. Messages include `GetVersionRequest`, `GetVersionResponse` (with `server BuildInfo=1; caller Principal=2`), `BuildInfo` (version, commit, built_at), and `Principal` (subject, provider). Provider enum: `PROVIDER_UNSPECIFIED=0; PROVIDER_GOOGLE=1; PROVIDER_AWS=2`.
- **Authorization**: Tokens with prefix `3am-aws-v1.` go to the AWS verifier; all others go to the Google verifier. The control plane never runs with an empty allowlist; an empty THREEAM_ALLOWED_PRINCIPALS makes the server exit non-zero at startup.

## Security Decisions and Why

- **No static keys**: Identity comes exclusively from Workload Identity Federation (GKE), Pod Identity (EKS), and GitHub OIDC. No service-account JSON keys, no IAM user access keys, no PATs in clusters. This eliminates the risk of key leakage and satisfies the founder's requirement that no billable credentials persist in repository source.
- **Fail-closed allowlist**: An empty `THREEAM_ALLOWED_PRINCIPALS` stops the server from starting. This ensures that a misconfigured deployment cannot accidentally accept all principals. The allowlist is populated only after founder approval.
- **Cosign KMS signing with `--tlog-upload=false`**: Signing uses a GCP KMS key so that the private key never leaves Google's infrastructure. The `--tlog-upload=false` flag prevents the public Rekor log from receiving the repo name and workflow names, which would expose private infrastructure details. Verification uses `--insecure-ignore-tlog=true` with the KMS public key, which is acceptable in the lab context where Rekor transparency is not required but cryptographic integrity is.
- **SSRF guard in AWS verifier**: Before any network call, the AWS verifier checks that the token's URL has scheme `https`, host matching `sts.amazonaws.com` or `sts.<region>.amazonaws.com`, path `/`, and query `Action=GetCallerIdentity` with `Version=2011-06-15`. It also verifies `X-Amz-SignedHeaders` contains `x-3am-audience` and `X-Amz-Expires` ≤ 60. Any failure rejects before any outbound request is made.
- **Public side channels**: Keyless cosign writes repo and workflow names to the public Rekor log, so the plans use a KMS key with `--tlog-upload=false`. The trade-off is that verification without a TLOG is insecure, but the `--insecure-ignore-tlog=true` flag is used only in lab and CI contexts where Rekor auditability is not required.
- **Evaluation honesty**: Truth is derived from the state the lab actually has, never asserted. Truth is sealed so only the scorer can open it, enforced at three levels: IAM, a Go import-boundary test, and a live PERMISSION_DENIED check. The seed is secret because determinism plus a known seed reveals the truth.
- **No AI attribution**: Plan files and commit messages carry no `Co-Authored-By` trailer and no "Generated with" line. This is a binding rule from the plan's ground rules.
- **Determinism**: Uses `math/rand/v2` PCG, a specified algorithm so the same seed gives the same output across machines and Go versions. Never calls `time.Now()` in generated data. Floats are rounded before hashing or ledgering.
- **Platform limits**: AWS `AssumeRole` ≥ 900 s, RDS auth tokens 15 min, GCP IAM propagation in minutes, open database sessions surviving revocation. Every duration is checked against platform limits.
- **Fleet-scale resources**: Per-database resources counted at 64 shards per server and 257 in total (replication slots, walsenders, processes, retained WAL), capped in aggregate.
- **Restores keep monotonic tokens**: Anything restored from a backup keeps fences and sequence numbers increasing (revision bump; ledger tail replay).

## How to Run Every Test and What Each Test Pins

### Cortex-Level Tests

- `terraform validate` in `terraform/envs/lab-aws`: `Success! The configuration is valid.` (exit code 0) — validates the AWS lab environment configuration. Pins: provider schema is correct; all variables have defaults or are marked required; backend configuration is syntactically valid.
- `terraform test` in `terraform/modules/aws-lab`: `Success! 5 passed, 0 failed.` — runs the 5 AWS lab module tests. Pins: pod_identity_association exists; threeam-cli role trusts pods.eks.amazonaws.com and sts:TagSession; CI role trusts exactly two repos (3am-control and 3am-cli on main); budget thresholds at 50%, 80%, 100%; no static key resources (aws_iam_access_key or aws_iam_user).
- `terraform fmt -check -recursive` (aws-lab): Pass (exit code 0) — checks Terraform formatting consistency. Pins: all .tf files use the project's formatting style.
- `go test ./...` in 3am-control: all packages pass (`ok`) — tests the auth interceptor, Google verifier, AWS STS verifier, config loader, version service, and controlplane server. Pins: interceptor rejects missing/unlisted tokens; Google verifier requires verified email and correct audience; AWS verifier rejects non-STS hosts, unsigned audience, long expiry, non-200 responses; config requires allowlist and audience; version service returns caller and server info; healthz works without auth.
- `go test ./...` in 3am-cli: all packages pass (`ok`) — tests token generation (Google AWS), version command (text and JSON output, endpoint behavior), and AWS token signing. Pins: AWS token signs audience, host, expires, signed headers; version without endpoint prints client only; version against fake server returns correct JSON; version server error exits non-zero.
- `buf lint` and `buf build` in 3am-proto (repo root): pass. `go build ./...` and `go vet ./...` in `3am-proto/gen/go`: pass. `go test ./...` at the 3am-proto root fails by design (no root Go module), so run it from `gen/go`.
- `go test ./...` in cortex root: runs across all managed repos' test suites as a cross-repo sanity check. Pins: no cross-repo contract violations.

### Individual Repo Tests (detailed in their own READMEs)

See each repo's README for task-specific test names, assertions, and pinned output.

## Acceptance Criteria and Their Current Status with Real Output

### Cortex-Level Acceptance

| Acceptance Clause | Status | Real Output |
|-------------------|--------|-------------|
| `terraform validate` in `terraform/envs/lab-aws` passes | ✅ Pass | `Success! The configuration is valid.` (exit code 0) |
| `terraform test` in `terraform/modules/aws-lab` passes (5 tests) | ✅ Pass | `Success! 5 passed, 0 failed.` |
| `terraform fmt -check -recursive` (aws-lab) passes | ✅ Pass | exit code 0 |
| `go test ./...` in 3am-control passes | ✅ Pass | all packages `ok` |
| `go test ./...` in 3am-cli passes | ✅ Pass | all packages `ok` |
| `buf lint` and `buf build` in 3am-proto pass | ✅ Pass | exit code 0 |
| `go build` and `go vet` in `3am-proto/gen/go` pass | ✅ Pass | exit code 0 |
| No `Co-Authored-By` or "Generated with" in any commit | ✅ Pass | no AI attribution added |

### Task-Level Acceptance (verified at commit time)

- **Task 7 step 6 commit** `4cd94b9` `feat(aws): lab skeleton with EKS Pod Identity, ECR, GitHub OIDC`: Terraform tests pass as shown above.
- **Task 8 commits** `d0db59c` (3am-control) and `57e3cf7` (3am-cli): Release workflows and .ko.yaml created; Go tests pass; no outward-facing actions (no push, no tag, no apply).
- **Task 8 Step 2** (cut v0.1.0 tag): Held pending founder approval — not executed.
- **Task 8 Step 3** (verify signatures): Held pending founder approval — not executed.
- **Task 8 Step 4** (verify SBOMs): Held pending founder approval — not executed.
- **Task 8 Step 5** (verify negative case): Held pending founder approval — not executed.

### Overall Step 0 Acceptance (verbatim from spec)

> "CLI ↔ API through Workload Identity / Pod Identity; signed images"

Status: **Not accepted.** Local work for Tasks 1–8 is committed. Task 7 step 5 (apply) and Task 8 steps 2–6 (tag, sign, verify, workflow runs) are not done. Task 9 (Helm acceptance) is not started. The acceptance sentence needs a live CLI-to-API call through Workload Identity and Pod Identity, plus signed images, and neither exists yet.

## Open Questions and Deferred Items

- **Monthly lab cap in USD (PLAN.md open question 12)**: No `terraform apply` in any step runs before the cap is set. The founder must set `monthly_cap_usd` before any cloud resource creation. This is a founder-approved prerequisite; no default value is assumed.
- **GCP project ID and AWS account ID**: Required before `terraform apply` in Tasks 6–7. Not set until founder provides them.
- **GCP KMS key for signing**: The SIGNING_KEY_KMS secret must be created in GitHub Actions. Not created until founder approval.
- **Repos under github.com/3am-si**: Done. The four private repos (3am-proto, 3am-control, 3am-cli, 3am-deploy) exist, and `main` is pushed to each.
- **PROTO_READ_TOKEN org Actions secret**: Fine-grained token with read-only contents on 3am-proto, used only in CI to fetch the private Go module. Must be set before CI runs.
- **Toolchain installation**: Go ≥ 1.26, buf, Terraform ≥ 1.9, ko, cosign, Helm 3, kubectl, AWS CLI, jq, kubeconform, and the Google Cloud CLI must be installed on Linux or macOS. Scripts run under bash 3.2 with no GNU-only flags.
- **AWS identity token design**: The AWS token copies the `aws-iam-authenticator` design. The token is a presigned `GetCallerIdentity` URL with a signed audience header. The exact design is deferred to the founder's security team.
- **Cedar authorization**: Arrives in step 7; not available in step 0. The allowlist is fail-closed without Cedar.
- **Cross-cloud calls**: Wait for the relay in step 3, not here. The control plane is deployed in both lab clusters, and each cluster's CLI calls its local server.
- **Helm chart linting and step-0 acceptance**: Task 9 deferred until after Task 8 is confirmed working with founder approval.
- **Terraform destroy and idle cost recording**: Task 9 includes `terraform destroy` for both envs and the recorded idle cost from GCP/AWS pricing pages. Deferred until after Tasks 6–8 stabilize.

**All open questions require founder input. No values are assumed or written into files.**

## Progress Snapshot

| Milestone | Item | State |
|-----------|------|-------|
| M1 step 0 | Tasks 1–6 | Committed and pushed |
| M1 step 0 | Task 7 (AWS lab) | Steps 1–4 done; apply (step 5) waiting on founder inputs |
| M1 step 0 | Task 8 (signed releases) | Local work committed; tag, sign, verify, and workflow runs waiting on founder approval |
| M1 step 0 | Task 9 (Helm acceptance) | Not started |
| M1 steps 1–13 | All | Not started |

Known issues: the release commits `8dba7cb` (3am-control) and `e839638` (3am-cli) are empty, because the release files were committed earlier in `d0db59c` and `57e3cf7`. Local branches `backup-before-rebuild` exist in 3am-control and 3am-cli and are not pushed.

