# M1 Step 0: Foundation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** A signed control-plane image serving one ConnectRPC call, and a `3am` CLI that calls it. The CLI authenticates with GKE Workload Identity in GCP and EKS Pod Identity in AWS, with no static keys anywhere.

**Architecture:** There are four new private repos under `github.com/3am-si`:
- `3am-proto` holds the API schema and the generated Go/Rust/TS/Python code.
- `3am-control` is the Go ConnectRPC server. Its auth interceptor verifies a Google ID token or a presigned AWS STS `GetCallerIdentity` request, then checks the caller against an allowlist.
- `3am-cli` is the Go `3am` binary.
- `3am-deploy` holds the Terraform skeletons for one GKE Autopilot and one EKS lab cluster, the registries, the KMS signing key, GitHub OIDC trust, a Helm chart and the end-to-end acceptance script.

**Tech Stack:** buf, Protobuf, ConnectRPC (connect-go, connect-es v2), Go ≥ 1.26, cobra, `google.golang.org/api/idtoken`, aws-sdk-go-v2, ko, cosign (GCP KMS key), Terraform ≥ 1.9, Helm 3, GitHub Actions.

**Spec:** `PLAN.md`: "Milestone 1 → Build order → step 0", "Founder decisions", "Repositories", "Cloud and on-prem integration → Placement and identity", "Console, CLI, API, Terraform → CLI".

## Global Constraints

- No static keys. Identity comes only from GKE Workload Identity Federation, EKS Pod Identity and GitHub OIDC. No service-account JSON keys, no IAM user access keys, no PATs in clusters.
- Go is used for the control plane, CLI and cloud. Rust and Python appear in this step only as generated-code compile checks.
- `3am-proto` generates clients for Go, Rust, TypeScript and Python.
- The API is ConnectRPC.
- Every image is signed with cosign and carries an SPDX SBOM attestation.
- The CLI is gcloud-style and composable over stdin/stdout. It supports `-o json`.
- All repos are private under `github.com/3am-si`. Go module paths: `github.com/3am-si/3am-proto/gen/go`, `github.com/3am-si/3am-control`, `github.com/3am-si/3am-cli`.
- The proto package is `threeam.meta.v1`. Resource-style APIs (`projects/{p}/locations/{site}/…`) start in later steps.
- Every commit message ends with `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`.
- Acceptance (verbatim from spec): "CLI ↔ API through Workload Identity / Pod Identity; signed images".

## Decisions made in this plan (not in the spec; founder to check)

1. **Images go to Artifact Registry (GCP) and ECR (AWS), not GHCR.** Clusters pull without static pull secrets. CI pushes via GitHub OIDC.
2. **Signing uses a GCP KMS key, with no upload to the public Rekor log.** Keyless signing would publish private repo and workflow names to public Rekor. Verification uses `--insecure-ignore-tlog=true` with the KMS public key.
3. **The control plane is deployed in both lab clusters, and each cluster's CLI calls its local server.** Cross-cloud calls go over the relay in step 3, not here.
4. **The allowlist is fail-closed.** An empty `THREEAM_ALLOWED_PRINCIPALS` stops the server from starting. Cedar authorization arrives in step 7.
5. **The AWS identity token copies the `aws-iam-authenticator` design.** The token is a presigned `GetCallerIdentity` URL with a signed audience header.
6. **Generated code is committed** in `3am-proto/gen/**`. CI fails if `buf generate` produces a diff.

## Review Focus

1. **A missing, empty or malformed `Authorization` header** gets `Unauthenticated`, and the handler never runs. Pinned in Task 2 (`TestInterceptorRejectsMissingToken`).
2. **An AWS token whose URL is not `https://sts[.<region>].amazonaws.com/?Action=GetCallerIdentity`** (SSRF to an internal host, `http://`, another action) is rejected *before any network call*. Pinned in Task 3 (`TestAWSRejectsNonSTSHostWithoutRequest`).
3. **A token minted for a different service**, meaning an AWS token without `x-3am-audience` in `X-Amz-SignedHeaders` or a Google ID token whose `aud` differs, is rejected. Pinned in Tasks 2 and 3.
4. **An empty allowlist** makes the server exit non-zero at startup and never allow everyone. Pinned in Task 4 (`TestLoadConfigRequiresAllowlist`).
5. **An EKS Pod Identity session ARN** `arn:aws:sts::123456789012:assumed-role/threeam-cli/eks-lab-threeam-cl-<uuid>` must match the allowlist entry `arn:aws:iam::123456789012:role/threeam-cli`, whatever the session name. Pinned in Task 3 (`TestNormalizeAssumedRoleARN`).

---

## Before you start (human-approved prerequisites)

These are outward-facing and cost money. **Ask the founder before each one.**

- [ ] Install the toolchain: `brew install go bufbuild/buf/buf hashicorp/tap/terraform ko cosign helm kubectl awscli jq kubeconform` and `brew install --cask google-cloud-sdk`. Check: `go version` reports ≥ 1.26 and `terraform version` reports ≥ 1.9.
- [ ] Create the private repos: `gh repo create 3am-si/3am-proto --private` (and the same for `3am-control`, `3am-cli`, `3am-deploy`). Clone them next to `cortex` in `~/Documents/3am/`.
- [ ] Get from the founder: the lab GCP project ID, the GCP region, the lab AWS account ID, the AWS region, and approval for the lab's running cost (PLAN.md open question 12).
- [ ] Create an org Actions secret `PROTO_READ_TOKEN`. It is a fine-grained token with read-only `contents` on `3am-proto`, used only in CI to fetch the private Go module. Set `GOPRIVATE=github.com/3am-si/*` in each Go repo's CI.

---

### Task 1: `3am-proto`: schema, codegen for four languages, CI

**Files:**
- Create: `3am-proto/buf.yaml`, `3am-proto/buf.gen.yaml`
- Create: `3am-proto/proto/threeam/meta/v1/version.proto`
- Create: `3am-proto/gen/go/go.mod` (module `github.com/3am-si/3am-proto/gen/go`)
- Create: `3am-proto/gen/rust/Cargo.toml` (crate `threeam-proto`), `3am-proto/gen/ts/package.json` (`@3am-si/proto`), `3am-proto/gen/python/pyproject.toml` (`threeam-proto`)
- Create: `3am-proto/.github/workflows/ci.yml`

**Interfaces:**
- Produces (Go): package `metav1` at `github.com/3am-si/3am-proto/gen/go/threeam/meta/v1`, and `metav1connect` with `NewVersionServiceHandler`, `NewVersionServiceClient` and `VersionServiceHandler`.
- Produces (messages): `GetVersionRequest{}`; `GetVersionResponse{server BuildInfo=1; caller Principal=2}`; `BuildInfo{version string=1; commit string=2; built_at google.protobuf.Timestamp=3}`; `Principal{subject string=1; provider Provider=2}`; `enum Provider{PROVIDER_UNSPECIFIED=0; PROVIDER_GOOGLE=1; PROVIDER_AWS=2}`.
- Produces (contract, written as a comment on `VersionService`): `Authorization: Bearer <token>`, where the token is either a Google ID token with `aud` = server audience, or `3am-aws-v1.` + base64url (no padding) of a presigned STS `GetCallerIdentity` GET URL whose signed headers include `x-3am-audience`.

- [ ] **Step 1: Write `version.proto`** with the messages and the single RPC `rpc GetVersion(GetVersionRequest) returns (GetVersionResponse)`. Add the auth contract comment above.
- [ ] **Step 2: Configure buf.** In `buf.yaml`: lint `STANDARD`, breaking `FILE`. In `buf.gen.yaml`, use remote plugins, each pinned to an explicit version: `protocolbuffers/go` + `connectrpc/go` → `gen/go` (`paths=source_relative`); `bufbuild/es` (`target=ts`) → `gen/ts/src`; `community/neoeinstein-prost` + `community/neoeinstein-prost-crate` → `gen/rust/src`; `protocolbuffers/python` + `protocolbuffers/pyi` → `gen/python/src`.
- [ ] **Step 3: Run lint, expect it to pass.** Run `buf lint`. Expected: no output, exit 0.
- [ ] **Step 4: Generate and compile all four languages.**
  Run `buf generate && (cd gen/go && go mod tidy && go build ./...) && (cd gen/rust && cargo build) && (cd gen/ts && npm i && npx tsc --noEmit) && (cd gen/python && python3 -I -c "import sys; sys.path.insert(0,'src'); import threeam.meta.v1.version_pb2")`.
  Expected: exit 0.
- [ ] **Step 5: Write CI** (`ci.yml`, on push and PR). It runs `buf lint`; then `buf breaking --against '.git#branch=main'` (PRs only); then `buf generate && git diff --exit-code`; then the four compile checks from Step 4.
- [ ] **Step 6: Commit, push, tag.** Commit `feat: threeam.meta.v1 VersionService and generated clients`. Push to `main`. Then `git tag gen/go/v0.1.0 && git push --tags`. Check: the CI run on `main` is green (`gh run watch`).

---

### Task 2: `3am-control`: auth interceptor and Google verifier

**Files:**
- Create: `3am-control/go.mod`
- Create: `3am-control/internal/auth/auth.go`, `3am-control/internal/auth/google.go`
- Test: `3am-control/internal/auth/auth_test.go`, `3am-control/internal/auth/google_test.go`

**Interfaces:**
- Consumes: `metav1.Provider` (Task 1).
- Produces:
  - `type Principal struct { Subject string; Provider metav1.Provider }`
  - `type Verifier interface { Verify(ctx context.Context, token string) (Principal, error) }`
  - `func NewMultiVerifier(google, aws Verifier) Verifier`. Tokens with prefix `3am-aws-v1.` go to `aws`; all others go to `google`. A nil verifier refuses tokens routed to it.
  - `func NewInterceptor(v Verifier, allowed map[string]bool) connect.Interceptor`. It returns `CodeUnauthenticated` for a missing or invalid token and `CodePermissionDenied` when `Subject` is not in `allowed`.
  - `func PrincipalFrom(ctx context.Context) (Principal, bool)`
  - `type ValidateFunc func(ctx context.Context, token, audience string) (*idtoken.Payload, error)`
  - `func NewGoogleVerifier(audience string, validate ValidateFunc) Verifier`. Production passes `idtoken.Validate`. `Subject` is the `email` claim, and the verifier requires `email_verified == true`.

- [ ] **Step 1: Write failing tests** with a stub `Verifier` and a connect test handler:
  - `TestInterceptorRejectsMissingToken`: no header, `"Bearer "` and `"Basic x"` each return `connect.CodeOf(err) == connect.CodeUnauthenticated`, and the handler call count stays 0.
  - `TestInterceptorRejectsUnlistedPrincipal`: a stub returns `{Subject: "x@y"}` and the allowlist is `{"a@b": true}`, so the result is `CodePermissionDenied`.
  - `TestInterceptorPutsPrincipalInContext`: in the handler, `PrincipalFrom(ctx)` returns the stub's principal.
  - `TestGoogleRejectsWrongAudience`: a `ValidateFunc` stub returns an error unless `audience == "3am-control"`. Verifying with `NewGoogleVerifier("other", stub)` errors.
  - `TestGoogleRequiresVerifiedEmail`: the payload has `email_verified: false`, so the verifier errors.
  - `TestMultiVerifierRoutesByPrefix`: `"3am-aws-v1.abc"` reaches the aws stub and `"eyJ..."` reaches the google stub.
- [ ] **Step 2: Run the tests, expect them to fail.** Run `go test ./internal/auth/...`. Expected: FAIL (undefined symbols).
- [ ] **Step 3: Implement** `auth.go` and `google.go` with the signatures above. The interceptor must also wrap streaming handlers (`WrapStreamingHandler`) with the same checks.
- [ ] **Step 4: Run the tests, expect them to pass.** Run `go test ./internal/auth/...`. Expected: PASS.
- [ ] **Step 5: Commit** `feat(auth): fail-closed interceptor and Google ID token verifier`.

---

### Task 3: `3am-control`: AWS STS verifier

**Files:**
- Create: `3am-control/internal/auth/aws.go`
- Test: `3am-control/internal/auth/aws_test.go`

**Interfaces:**
- Consumes: `Principal`, `Verifier` (Task 2).
- Produces:
  - `func NewAWSVerifier(audience string, client *http.Client) Verifier`. Production passes a client with a 5 s timeout.
  - `func NormalizeARN(arn string) (string, error)`. It maps `arn:aws:sts::<acct>:assumed-role/<role>/<session>` to `arn:aws:iam::<acct>:role/<role>`, and passes `arn:aws:iam::…` through unchanged.

**Verification order (before any network call, reject on the first failure):**
1. The token has prefix `3am-aws-v1.` and the rest base64url-decodes.
2. The URL parses with scheme `https`, and the host is `sts.amazonaws.com` or matches `^sts\.[a-z0-9-]+\.amazonaws\.com$`.
3. The path is `/` and the query has `Action=GetCallerIdentity` and `Version=2011-06-15`.
4. `X-Amz-SignedHeaders` contains `x-3am-audience`.
5. `X-Amz-Expires` ≤ 60.

Then send GET with headers `x-3am-audience: <audience>` and `Accept: application/json`. Any non-200 is an error. Read `GetCallerIdentityResponse.GetCallerIdentityResult.Arn`, apply `NormalizeARN`, and return `Principal{Subject: <normalized>, Provider: PROVIDER_AWS}`.

- [ ] **Step 1: Write failing tests** using a recording `http.RoundTripper` that returns canned STS JSON:
  - `TestAWSRejectsNonSTSHostWithoutRequest`: hosts `169.254.169.254`, `sts.amazonaws.com.evil.com` and `http://sts.amazonaws.com`, and `Action=AssumeRole`, each error with 0 round trips recorded.
  - `TestAWSRejectsUnsignedAudience`: `X-Amz-SignedHeaders=host` errors with 0 round trips.
  - `TestAWSRejectsLongExpiry`: `X-Amz-Expires=900` errors.
  - `TestAWSSendsAudienceHeader`: the recorded request has `x-3am-audience: 3am-control`.
  - `TestAWSNon200IsUnauthenticated`: canned 403 makes the verifier error.
  - `TestNormalizeAssumedRoleARN`: `arn:aws:sts::123456789012:assumed-role/threeam-cli/eks-lab-threeam-cl-1f2e` → `arn:aws:iam::123456789012:role/threeam-cli`; `arn:aws:iam::123456789012:role/threeam-cli` unchanged; `"nope"` errors.
- [ ] **Step 2: Run the tests, expect them to fail.** Run `go test ./internal/auth/ -run 'AWS|Normalize'`. Expected: FAIL.
- [ ] **Step 3: Implement `aws.go`** in the verification order above.
- [ ] **Step 4: Run the tests, expect them to pass.** Run `go test ./internal/auth/...`. Expected: PASS.
- [ ] **Step 5: Commit** `feat(auth): AWS STS GetCallerIdentity verifier with SSRF guard`.

---

### Task 4: `3am-control`: VersionService and server binary

**Files:**
- Create: `3am-control/internal/buildinfo/buildinfo.go`
- Create: `3am-control/internal/version/service.go`
- Create: `3am-control/internal/config/config.go`
- Create: `3am-control/cmd/controlplane/main.go`
- Create: `3am-control/.github/workflows/ci.yml`
- Test: `3am-control/internal/version/service_test.go`, `3am-control/internal/config/config_test.go`

**Interfaces:**
- Consumes: `auth.NewInterceptor`, `auth.NewMultiVerifier`, `auth.NewGoogleVerifier`, `auth.NewAWSVerifier`, `auth.PrincipalFrom` (Tasks 2–3); `metav1connect.NewVersionServiceHandler` (Task 1).
- Produces:
  - `buildinfo.Version`, `buildinfo.Commit`, `buildinfo.BuiltAt` (string vars set by `-ldflags -X`; defaults `"dev"`, `"none"`, `""`).
  - `type version.Service struct{}` implementing `metav1connect.VersionServiceHandler`. It returns `server` from buildinfo and `caller` from `PrincipalFrom`.
  - `type config.Config struct { Listen, Audience string; Allowed map[string]bool; Providers []string }`
  - `func config.Load(getenv func(string) string) (Config, error)`. It reads `THREEAM_LISTEN` (default `:8080`), `THREEAM_AUDIENCE` (required), `THREEAM_ALLOWED_PRINCIPALS` (comma-separated, required, non-empty) and `THREEAM_AUTH_PROVIDERS` (comma-separated subset of `google,aws`, default `google,aws`).
  - HTTP behaviour: `GET /healthz` returns 200 `ok` with no auth. The Connect handler is mounted with the interceptor and served over h2c.

- [ ] **Step 1: Write failing tests:**
  - `TestLoadConfigRequiresAllowlist`: `THREEAM_ALLOWED_PRINCIPALS` set to `""` or `" , "` returns an error.
  - `TestLoadConfigRequiresAudience`: an empty audience returns an error.
  - `TestLoadConfigRejectsUnknownProvider`: `THREEAM_AUTH_PROVIDERS=google,azure` returns an error.
  - `TestGetVersionReturnsCaller`: an `httptest` server with interceptor + stub verifier `{Subject:"cli@p.iam.gserviceaccount.com", Provider: PROVIDER_GOOGLE}`, called through the generated client with a token. The response has `caller.subject == "cli@p.iam.gserviceaccount.com"` and `server.version == buildinfo.Version`.
  - `TestHealthzNoAuth`: `GET /healthz` without a token returns 200.
- [ ] **Step 2: Run the tests, expect them to fail.** Run `go test ./...`. Expected: FAIL.
- [ ] **Step 3: Implement** `buildinfo`, `config`, `version.Service` and `main.go`. In `main.go`, a `config.Load` error is logged to stderr with exit 1.
- [ ] **Step 4: Run the tests, expect them to pass.** Run `go test ./... && go vet ./...`. Expected: PASS.
- [ ] **Step 5: Run a manual smoke test.** Run `THREEAM_AUDIENCE=3am-control go run ./cmd/controlplane`. Expected: exit 1 with an error mentioning `THREEAM_ALLOWED_PRINCIPALS`.
- [ ] **Step 6: Write CI** (`go test ./...`, `go vet`, `staticcheck`, `GOPRIVATE` + `PROTO_READ_TOKEN`) **and commit** `feat: VersionService and controlplane server`.

---

### Task 5: `3am-cli`: `3am version`

**Files:**
- Create: `3am-cli/go.mod`, `3am-cli/cmd/3am/main.go`
- Create: `3am-cli/internal/buildinfo/buildinfo.go` (same three vars as Task 4)
- Create: `3am-cli/internal/token/google.go`, `3am-cli/internal/token/aws.go`
- Create: `3am-cli/internal/cmd/root.go`, `3am-cli/internal/cmd/version.go`
- Create: `3am-cli/.github/workflows/ci.yml`
- Test: `3am-cli/internal/token/aws_test.go`, `3am-cli/internal/cmd/version_test.go`

**Interfaces:**
- Consumes: `metav1connect.NewVersionServiceClient` (Task 1). The token contract is in the Task 1 proto comment.
- Produces:
  - `func token.Google(ctx context.Context, audience string) (string, error)`. Uses `idtoken.NewTokenSource(ctx, audience)` and returns `Token().AccessToken`.
  - `func token.AWS(ctx context.Context, cfg aws.Config, audience string) (string, error)`. Presigns `sts:GetCallerIdentity` with header `x-3am-audience: <audience>` added to the signed headers and expiry 60 s. Returns `"3am-aws-v1." + base64.RawURLEncoding.EncodeToString(url)`.
  - `func cmd.NewRoot(out, errOut io.Writer) *cobra.Command`. Persistent flags: `--endpoint` (env `THREEAM_ENDPOINT`), `--auth` (`google|aws|none`, env `THREEAM_AUTH`, default `none`), `--audience` (env `THREEAM_AUDIENCE`, default `3am-control`), `-o/--output` (`text|json`, default `text`).
  - Output of `3am version`, text form:
    ```
    client  <version> (<commit>)
    server  <version> (<commit>)
    caller  <subject> (<google|aws>)
    ```
  - Output of `3am version`, `-o json` form: `{"client":{"version","commit"},"server":{"version","commit"},"caller":{"subject","provider"}}`.
  - Behaviour with no `--endpoint`: print only the client and exit 0.
  - Behaviour when the server call fails: print the client to `out`, print `error: <connect code>: <message>` to `errOut`, and exit 1.

- [ ] **Step 1: Write failing tests:**
  - `TestAWSTokenSignsAudience`: static test creds (`credentials.NewStaticCredentialsProvider("AKIDEXAMPLE","secret","")`, region `us-east-1`). Decoding the token gives a URL with host `sts.us-east-1.amazonaws.com`, `Action=GetCallerIdentity`, `X-Amz-Expires=60`, and `X-Amz-SignedHeaders` containing `x-3am-audience`. These creds are test-only and never leave the test.
  - `TestVersionJSONAgainstFakeServer`: an `httptest` server running a `VersionServiceHandler` stub returns server `v9.9.9` and caller `cli@p`. With `-o json --endpoint <url>`, the parsed JSON has `.server.version == "v9.9.9"` and `.caller.subject == "cli@p"`.
  - `TestVersionWithoutEndpointPrintsClientOnly`: output has one line starting `client`, with exit 0.
  - `TestVersionServerErrorExitsNonZero`: the stub returns `CodeUnauthenticated`, so the command errors and `errOut` contains `unauthenticated`.
- [ ] **Step 2: Run the tests, expect them to fail.** Run `go test ./...`. Expected: FAIL.
- [ ] **Step 3: Implement** the token functions and commands with the signatures above. Use `config.LoadDefaultConfig(ctx)` in production. It picks up Pod Identity through the container credential provider.
- [ ] **Step 4: Run the tests, expect them to pass.** Run `go test ./... && go vet ./...`. Expected: PASS.
- [ ] **Step 5: Run a manual check.** Run `go run ./cmd/3am version`. Expected: `client  dev (none)`.
- [ ] **Step 6: Write CI** (as in Task 4) **and commit** `feat: 3am version with Google and AWS workload identity`.

---

### Task 6: `3am-deploy`: GCP lab skeleton

**Files:**
- Create: `3am-deploy/terraform/modules/gcp-lab/{main.tf,variables.tf,outputs.tf,versions.tf}`
- Create: `3am-deploy/terraform/envs/lab-gcp/{main.tf,backend.tf,terraform.tfvars.example}`
- Create: `3am-deploy/scripts/bootstrap-state.sh`
- Test: `3am-deploy/terraform/modules/gcp-lab/tests/gcp_lab.tftest.hcl`

**Interfaces:**
- Inputs: `project_id`, `region`, `github_org` (default `3am-si`).
- Resources:
  - GKE Autopilot cluster `threeam-lab`.
  - GSAs `threeam-cli` and `threeam-control`.
  - `roles/iam.workloadIdentityUser` on `threeam-cli` for member `serviceAccount:<project>.svc.id.goog[threeam/threeam-cli]`.
  - Artifact Registry docker repo `threeam`.
  - KMS keyring `threeam` with key `image-signing` (`ASYMMETRIC_SIGN`, `EC_SIGN_P256_SHA256`).
  - A Workload Identity Pool `github` with an OIDC provider for `https://token.actions.githubusercontent.com`. Its attribute condition is `assertion.repository_owner == "<github_org>"`.
  - A CI service account `threeam-ci` with `artifactregistry.writer` on the repo and `cloudkms.signerVerifier` on the key. It is impersonable only by `principalSet://…/attribute.repository/3am-si/3am-control` and `…/3am-si/3am-cli`.
- Outputs: `cluster_name`, `registry` (`<region>-docker.pkg.dev/<project>/threeam`), `cli_principal` (GSA email), `signing_key` (`gcpkms://projects/…/cryptoKeyVersions/1`), `ci_wif_provider`, `ci_service_account`.
- Backend: GCS bucket `<project>-threeam-tfstate`, created by `bootstrap-state.sh gcp <project> <region>` with versioning on.

- [ ] **Step 1: Write a failing `terraform test`** with `mock_provider "google" {}`. Assert:
  - the WI binding member equals `"serviceAccount:test-proj.svc.id.goog[threeam/threeam-cli]"`;
  - the pool provider `attribute_condition` contains `repository_owner == "3am-si"`;
  - the KMS key `purpose == "ASYMMETRIC_SIGN"`;
  - no `google_service_account_key` resource exists.
- [ ] **Step 2: Run it, expect it to fail.** Run `terraform -chdir=terraform/modules/gcp-lab init && terraform -chdir=terraform/modules/gcp-lab test`. Expected: FAIL.
- [ ] **Step 3: Implement the module and the `lab-gcp` env.**
- [ ] **Step 4: Run it, expect it to pass.** Run `terraform fmt -check -recursive && terraform -chdir=terraform/modules/gcp-lab test && terraform -chdir=terraform/envs/lab-gcp validate`. Expected: PASS.
- [ ] **Step 5: Apply (founder approval first; costs money).** Run `scripts/bootstrap-state.sh gcp …` and then `terraform -chdir=terraform/envs/lab-gcp apply`. Check: `gcloud container clusters get-credentials threeam-lab --region <region>` succeeds.
- [ ] **Step 6: Commit** `feat(gcp): lab skeleton with Workload Identity, registry, KMS signing key, GitHub OIDC`.

---

### Task 7: `3am-deploy`: AWS lab skeleton

**Files:**
- Create: `3am-deploy/terraform/modules/aws-lab/{main.tf,variables.tf,outputs.tf,versions.tf}`
- Create: `3am-deploy/terraform/envs/lab-aws/{main.tf,backend.tf,terraform.tfvars.example}`
- Modify: `3am-deploy/scripts/bootstrap-state.sh` (add `aws <account> <region>`: S3 bucket `<account>-threeam-tfstate`, versioned, with S3 native locking)
- Test: `3am-deploy/terraform/modules/aws-lab/tests/aws_lab.tftest.hcl`

**Interfaces:**
- Inputs: `region`, `github_org` (default `3am-si`).
- Resources:
  - VPC via `terraform-aws-modules/vpc/aws` (3 AZs, a single NAT gateway to keep lab cost down).
  - EKS `threeam-lab` via `terraform-aws-modules/eks/aws`, with addon `eks-pod-identity-agent` and one managed node group (2 × `t3.medium`).
  - IAM role `threeam-cli`. Its trust policy has principal `pods.eks.amazonaws.com` with actions `sts:AssumeRole` and `sts:TagSession`. It has no policies attached; `GetCallerIdentity` needs none.
  - `aws_eks_pod_identity_association` for namespace `threeam`, service account `threeam-cli`.
  - ECR repos `threeam/threeam-control` and `threeam/3am-cli` (immutable tags).
  - GitHub OIDC provider and role `threeam-ci`, trusted for `repo:3am-si/3am-control:ref:refs/heads/main` and `repo:3am-si/3am-cli:ref:refs/heads/main`. It may only push to those two ECR repos.
- Outputs: `cluster_name`, `registry` (`<account>.dkr.ecr.<region>.amazonaws.com/threeam`), `cli_principal` (`arn:aws:iam::<account>:role/threeam-cli`), `ci_role_arn`.

- [ ] **Step 1: Write a failing `terraform test`** with `mock_provider "aws"`. Assert:
  - the pod identity association `namespace == "threeam"` and `service_account == "threeam-cli"`;
  - the `threeam-cli` trust policy JSON contains `pods.eks.amazonaws.com` and `sts:TagSession`;
  - the `threeam-ci` trust `sub` condition lists exactly the two repos above;
  - no `aws_iam_access_key` or `aws_iam_user` resource exists.
- [ ] **Step 2: Run it, expect it to fail.** Run `terraform -chdir=terraform/modules/aws-lab init && terraform -chdir=terraform/modules/aws-lab test`. Expected: FAIL.
- [ ] **Step 3: Implement the module and the `lab-aws` env.**
- [ ] **Step 4: Run it, expect it to pass.** Run `terraform fmt -check -recursive && terraform -chdir=terraform/modules/aws-lab test && terraform -chdir=terraform/envs/lab-aws validate`. Expected: PASS.
- [ ] **Step 5: Apply (founder approval first).** Check: `aws eks update-kubeconfig --name threeam-lab` succeeds.
- [ ] **Step 6: Commit** `feat(aws): lab skeleton with EKS Pod Identity, ECR, GitHub OIDC`.

---

### Task 8: Signed releases for `3am-control` and `3am-cli`

**Files:**
- Create: `3am-control/.ko.yaml`, `3am-control/.github/workflows/release.yml`
- Create: `3am-cli/.ko.yaml`, `3am-cli/.github/workflows/release.yml`

**Interfaces:**
- Consumes: Task 6 outputs `registry`, `signing_key`, `ci_wif_provider`, `ci_service_account`; Task 7 outputs `registry`, `ci_role_arn`. Store them as repo Actions *variables*, not secrets, since none of them is secret.
- Produces: images `<gcp registry>/threeam-control@sha256:…` and `<gcp registry>/3am-cli@sha256:…`, copied to the same names under the ECR registry. Each is signed with `signing_key` and has an `spdxjson` attestation. Each workflow run writes the digest to its job summary.

**Workflow** (on tag `v*`; `permissions: id-token: write, contents: read`):
1. Run `google-github-actions/auth` (WIF).
2. Run `ko build ./cmd/<bin> --bare --sbom=spdx --sbom-dir=sbom --tags=<tag>`, with ldflags setting `buildinfo.Version=<tag>`, `Commit=<sha>` and `BuiltAt=<RFC3339>`. Base image: ko's default distroless static.
3. Run `cosign sign --key $SIGNING_KEY --tlog-upload=false <digest>`.
4. Run `cosign attest --key $SIGNING_KEY --tlog-upload=false --type spdxjson --predicate sbom/*.spdx.json <digest>`.
5. Run `aws-actions/configure-aws-credentials` (OIDC role), then `cosign copy <gcp digest> <ecr repo>`. This also copies signatures and attestations.

- [ ] **Step 1: Write both workflows and `.ko.yaml` files.**
- [ ] **Step 2: Cut `v0.1.0` in both repos.** Run `git tag v0.1.0 && git push --tags`. Check: `gh run watch` is green in both.
- [ ] **Step 3: Verify the signatures from your laptop.** Run `cosign verify --key <signing_key> --insecure-ignore-tlog=true <each of the 4 image digests>`. Expected: `Verification for … The cosign claims were validated` ×4.
- [ ] **Step 4: Verify the SBOMs.** Run `cosign verify-attestation --key <signing_key> --insecure-ignore-tlog=true --type spdxjson <digest> | jq -r .payload | base64 -d | jq .predicateType`. Expected: `"https://spdx.dev/Document"`.
- [ ] **Step 5: Verify the negative case.** Run `cosign verify` with a freshly generated `cosign generate-key-pair` key. Expected: non-zero exit.
- [ ] **Step 6: Commit both repos** `ci: ko build, KMS-signed images with SPDX SBOM`.

---

### Task 9: Helm chart and step-0 acceptance run

**Files:**
- Create: `3am-deploy/charts/threeam-control/{Chart.yaml,values.yaml,templates/{deployment,service,serviceaccount}.yaml}`
- Create: `3am-deploy/k8s/cli-smoke-job.yaml.tmpl`
- Create: `3am-deploy/scripts/e2e-step0.sh`
- Create: `3am-deploy/README.md` (lab bring-up, teardown, and the idle cost of both clusters taken from the GCP/AWS pricing pages)

**Interfaces:**
- Consumes: image digests (Task 8); `cli_principal` outputs (Tasks 6–7); env var contract (Task 4); CLI flags and JSON shape (Task 5).
- Chart values:
  - `image` (digest required; the template fails if it has no `@sha256:`);
  - `audience` (default `3am-control`);
  - `allowedPrincipals` (list, required);
  - `authProviders` (default `[google, aws]`);
  - `serviceAccount.annotations`.
  The Service is ClusterIP `threeam-control:8080` in namespace `threeam`. Liveness and readiness probes hit `/healthz`.
- Smoke Job: it runs as KSA `threeam-cli`. On GKE that KSA is annotated `iam.gke.io/gcp-service-account: <cli_principal>`. The Job runs `3am version --endpoint http://threeam-control.threeam.svc:8080 --auth {google|aws} -o json`.
- `e2e-step0.sh <gke-context> <eks-context>`: for each cloud it installs the chart, runs the Job, then asserts and verifies as in Step 3. It exits non-zero on any failure.

- [ ] **Step 1: Lint the chart and check that a digest is required.**
  - Run `helm lint charts/threeam-control --set image=x@sha256:$(printf '0%.0s' {1..64}) --set allowedPrincipals={a}`. Expected: `1 chart(s) linted, 0 chart(s) failed`.
  - Run `helm template charts/threeam-control --set image=x:latest --set allowedPrincipals={a}`. Expected: an error mentioning `@sha256`.
- [ ] **Step 2: Validate the rendered manifests.** Run `helm template … | kubeconform -strict -summary`. Expected: `Invalid: 0, Errors: 0`.
- [ ] **Step 3: Write `e2e-step0.sh`.** For each of GKE and EKS, it must:
  - (a) run `cosign verify` on the digest it deploys;
  - (b) run `helm upgrade --install` with `allowedPrincipals={<that cloud's cli_principal>}`;
  - (c) run the smoke Job and `kubectl wait --for=condition=complete`;
  - (d) parse the logs with `jq` and assert `.caller.subject == <cli_principal>`, `.caller.provider` (`google` or `aws`), and `.server.version == "v0.1.0"`;
  - (e) **as a negative check**, run the same Job as KSA `default` (no identity) and assert it fails with `unauthenticated` in its logs.
- [ ] **Step 4: Run the acceptance check.** Run `scripts/e2e-step0.sh <gke-ctx> <eks-ctx>`. Expected: four `PASS` lines (GKE positive, GKE negative, EKS positive, EKS negative), then `STEP 0 ACCEPTED`, exit 0.
- [ ] **Step 5: Write the README**, including `terraform destroy` for both envs and the recorded idle cost.
- [ ] **Step 6: Commit** `feat: control-plane chart and step-0 acceptance script`. Paste the Step 4 output into the PR or commit body as evidence.
- [ ] **Step 7: Ask the founder** whether to keep the lab clusters running for step 1 or `terraform destroy` them now.
