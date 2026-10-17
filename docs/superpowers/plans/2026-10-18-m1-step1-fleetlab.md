# M1 Step 1: Fleetlab Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build `3am-fleetlab`:
- a 257-shard payroll lab across GCP, AWS and an isolated "on-prem" VPC;
- a variant catalogue;
- a deterministic injector;
- a scorer that is the only reader of the sealed ground truth.

**Architecture:** One Go module, one binary `fleetlab`.
- **Faults are causal, not painted.** The injector changes the *rollout state* (which release runs on which shard, which tenants have which flag). The lab's `pay` and `attendance` workers carry simulated releases with real bugs, so bad rows appear through normal writes.
- **Truth is a pure function** of (variant spec, seed, baseline). It is sealed at birth with envelope encryption to a KMS RSA key that only the scorer identity can decrypt.
- **Each site runs a `fleetlab site-agent`.** It pulls orders from an outbound-only queue (GCS/S3), so the on-prem VPC needs no inbound rule.

**Tech Stack:** Go ≥ 1.26 (`math/rand/v2` PCG for seeded RNG), pgx v5, `fergusstrange/embedded-postgres` for tests (no Docker), `gocloud.dev/blob` (gcsblob, s3blob, fileblob), Cloud KMS, Terraform ≥ 1.9, Cloud SQL / RDS / Compute Engine Postgres, ko + cosign (the Step 0 pipeline).

**Spec:** `PLAN.md`: "Milestone 1 → Lab (`3am-fleetlab`)", "Variant catalogue", "Build order step 1"; "Learning and trust → Evaluation discipline"; "Repositories".

**Depends on:** Step 0 plan `docs/superpowers/plans/2026-10-18-m1-step0-foundation.md`. That plan provides the GKE/EKS lab clusters, registries, the KMS image-signing key and GitHub OIDC. This plan reads them through `terraform_remote_state` on the Step 0 state buckets.

## Global Constraints

- Acceptance (verbatim from spec): "`inject <variant> --seed N` is deterministic; the scorer is the only reader of the truth; idle cost documented".
- Shard count: **257** logical Postgres databases. The split is GCP ~128, AWS ~64, on-prem ~65, with **no inbound rules** on-prem. Each site has primaries plus a replica.
- Payroll schema tables: `tenants, employees, timecards, pay_runs, pay_lines, accruals, gl_entries`. `gl_entries` is ledger-style: append-only, corrected by compensating entries. There is also a routing directory.
- The faulty release is `2026.10.3`, and the faulty flag is `accrual_v2`.
- The flag service buckets tenants by hash.
- "The fleetlab ground truth is **vaulted**: the scorer holds it, and no model or pack sees it."
- Results are reported with Clopper–Pearson intervals, or the rule of three on 0 failures, so "0/200" reads ≤ 1.5% at 95%.
- Idle instances are stopped.
- No static keys: the same rule as Step 0. On-prem VMs use an attached service account; the pods use WI/Pod Identity.
- Every image is signed by the Step 0 release pipeline.
- Module `github.com/3am-si/3am-fleetlab`, private.
- Every commit message ends with `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`.

## Decisions made in this plan (not in the spec; founder to check)

1. **Shard placement** is fixed. Engine and replica placement is the only way to make `+no-overlap` satisfiable, since it needs "every release-R shard is v15, no clean shard is":

   | Server | Site | Engine | Shards |
   |---|---|---|---|
   | `gcp-sql-a` | gcp (Cloud SQL) | PG 17 | `shard-000`–`shard-063` |
   | `gcp-sql-b` | gcp (Cloud SQL) | PG 17 | `shard-064`–`shard-127` |
   | `aws-rds-a` | aws (RDS) | PG 17 | `shard-128`–`shard-191` |
   | `onprem-a` | onprem (GCE VM) | **PG 15** | `shard-192`–`shard-229` (38) |
   | `onprem-b` | onprem (GCE VM) | PG 17 | `shard-230`–`shard-256` (27) |

   Each server has one streaming/read replica. The routing directory lives in database `directory` on `gcp-sql-a`.
2. **The "on-prem" site is a GCP VPC** with zero ingress firewall rules (intra-VPC replication allowed) and Cloud NAT for egress only.
3. **`pay` and `attendance` run at every site**: GKE for gcp, EKS for aws, a container on an on-prem VM for onprem. The spec says "on GKE Autopilot", but GKE cannot reach an on-prem VPC that has no inbound rules.
4. **Simulated releases live in one binary.** The worker reads its shard's release from the directory and dispatches to `release_2026_10_2`, `release_2026_10_3` (bug) or `release_2026_10_3_fixed`.
5. **Seven variants are catalogued now but injectable only in a later step**, because they need components that don't exist yet. `fleetlab inject` refuses them with `variant +X is available from step N`:

   | Variant | Available from step |
   |---|---|
   | `+pg-resurrect` | 4 |
   | `+leap-smear` | 4 |
   | `+sqlserver-skip` | 4 |
   | `+failover` | 8 |
   | `+late-report` | 8 |
   | `+relay-loss` | 8 |
   | `+evidence-injection` | 13 |

6. **The seed is secret.** Determinism means anyone holding (spec, seed) can recompute the truth. So the seed lives only inside the sealed truth and the operator's command line. Run records, orders, logs and DB rows never contain it.
7. **`fleetlab score --reveal`** prints the per-shard diff, ledgers a `reveal` event and marks the run *burned*. Burned runs are excluded from every later evaluation report.

## Open questions this plan surfaces (founder)

- **`+sqlserver-skip` needs a SQL Server shard**, but the M1 lab is all Postgres, and SQL Server is deferred to M2 "unless UKG is SQL Server". The catalogue keeps the variant with `available_from_step: 4` and `requires_engine: sqlserver`. Should we add one SQL Server instance in M1?
- The lab's running cost needs approval (PLAN.md open question 12). Task 11 measures it.

## Review Focus

1. **The truth must match reality.** The truth's `BadRows` per (shard, tenant) must equal the rows the lab actually contains after `traffic --batch`. A truth that disagrees with the DB scores every system wrongly. Pinned in Task 12 (`fleetlab verify-lab`) and Task 6 (`TestTruthBadRowsMatchSimulatedWrites`).
2. **No tells in the data.** Injected effects must not be distinguishable by `application_name`, role, SQL comment or a column only the injector writes. Pinned in Task 6 (`TestNoInjectorTells`).
3. **Same seed, different machine or Go version.** Plans must be byte-identical. Pinned by a golden-hash file committed in Task 6 (`TestPlanGoldenHashes`), which fails on any drift.
4. **Conflicting modifiers** (`+null+bleeding`, `+randomised+threshold`) must be refused, not silently merged. Pinned in Task 5 (`TestConflictingModifiersRefused`).
5. **RDS auto-restarts a stopped instance after 7 days**, breaking "idle instances are stopped". Pinned in Task 11 by an EventBridge Scheduler rule that re-stops it daily, with a tftest asserting the rule exists.

---

## File structure (`3am-fleetlab/`)

| Path | Responsibility |
|---|---|
| `cmd/fleetlab/main.go` | Subcommands: `catalogue`, `reset`, `inject`, `traffic`, `checksum`, `score`, `verify-lab`, `lab up/down/status`, `site-agent` |
| `internal/topology/` | Static servers and shards (Decision 1) |
| `internal/schema/` | Embedded SQL: `payroll.sql` (per shard), `directory.sql` |
| `internal/baseline/` | Seeded per-shard dataset generator (pure) |
| `internal/rollout/` | Directory model: releases per shard, flags with `hash|list|threshold` rules; `Enabled` evaluation |
| `internal/pay/`, `internal/attendance/` | Workers with simulated releases |
| `internal/traffic/` | Seeded batch schedule driving the workers |
| `internal/variant/` | Catalogue, spec parser, composition, one file per variant |
| `internal/plan/` | `Build` = spec + seed → `Plan` (orders) + `Truth` (pure) |
| `internal/truth/` | `Truth` type, `Seal` (public key only) |
| `internal/vault/` | `Open` (KMS decrypt). **Imported only by `internal/scorer` and `cmd/fleetlab`'s score and verify-lab commands** |
| `internal/scorer/` | `Score`, `Aggregate`, Clopper–Pearson |
| `internal/agent/` | Site agent: order queue, apply, ack, checksum |
| `terraform/gcp`, `terraform/aws` | Lab infra |
| `deploy/` | Helm values and the on-prem VM container spec for the agent and workers |
| `scripts/e2e-step1.sh` | Acceptance |

---

### Task 1: Repo, topology, baseline generator

**Files:**
- Create: `go.mod`, `internal/topology/topology.go`, `internal/baseline/baseline.go`
- Test: `internal/topology/topology_test.go`, `internal/baseline/baseline_test.go`

**Interfaces:**
- Produces:
  - `type Site string` (`"gcp"`, `"aws"`, `"onprem"`);
  - `type Server struct{ ID string; Site Site; PGMajor int }`;
  - `type Shard struct{ ID string; Index int; Server Server }`;
  - `func topology.Shards() []Shard` (257, in index order);
  - `func topology.Servers() []Server`;
  - `type Dataset struct{ Tenants []Tenant; Employees []Employee; Timecards []Timecard }`, with exported row structs mirroring the schema columns in Task 2;
  - `func baseline.Generate(seed uint64, s topology.Shard) Dataset`.
- Baseline values:
  - Tenant IDs are `int64(s.Index)*1000 + k` for `k ∈ [1, 40]`.
  - Each tenant has 20–80 employees, drawn uniformly.
  - Timecards cover 28 days from `2026-09-01`, one per employee per weekday, with hours drawn from `{7.5, 8, 8.5}`.
  - The RNG is `rand.New(rand.NewPCG(seed, uint64(s.Index)))`.
  - No `time.Now()` anywhere.

- [ ] **Step 1: Write failing tests:**
  - `TestShardCountAndPlacement`: `len(Shards()) == 257`. Counts per site are gcp 128, aws 64, onprem 65. `shard-192` and `shard-229` are on `onprem-a` with `PGMajor == 15`.
  - `TestBaselineDeterministic`: `Generate(7, s)` called twice is deep-equal. `Generate(8, s)` differs.
  - `TestBaselineTenantIDsUnique`: tenant IDs are unique across all 257 shards for seed 1.
- [ ] **Step 2: Run, expect failure.** Run `go test ./internal/...`. Expected: FAIL.
- [ ] **Step 3: Implement** with the signatures above.
- [ ] **Step 4: Run, expect pass.** Run `go test ./internal/...`. Expected: PASS.
- [ ] **Step 5: Create the repo and commit.** Get founder approval, then run `gh repo create 3am-si/3am-fleetlab --private`. Commit `feat: topology and seeded baseline generator`.

### Task 2: Schema, directory, flag evaluation

**Files:**
- Create: `internal/schema/payroll.sql`, `internal/schema/directory.sql`, `internal/schema/schema.go` (embed), `internal/rollout/rollout.go`
- Test: `internal/rollout/rollout_test.go`, `internal/schema/schema_test.go`

**Interfaces:**
- `payroll.sql` (per shard):
  - `tenants(tenant_id bigint pk, name text, region text)`;
  - `employees(employee_id bigint pk, tenant_id bigint not null references tenants)`;
  - `timecards(tenant_id, employee_id, work_date date, hours numeric(4,2), pk(tenant_id, employee_id, work_date))`;
  - `pay_runs(pay_run_id bigint pk, tenant_id, period_start date, period_end date, status text)`;
  - `pay_lines(pay_run_id, employee_id, tenant_id, gross numeric(12,2))`;
  - `accruals(tenant_id, employee_id, pay_run_id, hours_accrued numeric(6,2))`;
  - `gl_entries(entry_id bigint pk, tenant_id, pay_run_id, account text, amount numeric(14,2), reverses bigint null)`, append-only: a trigger rejects `UPDATE` and `DELETE`;
  - `timecards_dup_log(tenant_id, employee_id, work_date, hours)`, the `+decoy` symptom table;
  - `job_runs(tenant_id, pay_run_id, job text, ran bool)`, the nightly accrual-reconcile job record for `+post-treatment-trap`;
  - roles `payroll_app` (read/write) and `threeam_reader` (read-only, used from Step 3).
- `directory.sql`:
  - `shards(shard_id text pk, server text, site text, routing_version int)`;
  - `tenant_shard(tenant_id bigint, shard_id text, version int, pk(tenant_id, version))`;
  - `releases(shard_id, service text, release text, effective_seq bigint)`;
  - `flags(flag text pk, rule text check (rule in ('hash','list','threshold')), salt text, pct numeric, cutoff bigint, tenant_list bigint[])`;
  - `shard_attributes(shard_id, name text, value text)`, used by `+knockoff-calibration` and the co-rolled flag.
- Produces: `func rollout.Enabled(f Flag, tenantID int64) bool`. For `hash`, it computes FNV-1a 64 of `salt + ":" + decimal(tenantID)`, takes the result mod 10000, and compares `< int(pct*100)`. For `threshold`, it is `tenantID < cutoff`. For `list`, it is membership. Also `func rollout.ReleaseOf(shardID, service string, seq int64) string`.

- [ ] **Step 1: Write failing tests:**
  - `TestHashFlagStable`: `Enabled({rule:"hash", salt:"accrual_v2", pct:10}, 1001)` returns a hard-coded value, recorded once and committed.
  - `TestHashFlagRate`: across 100k tenant IDs, `pct:10` enables 10% ± 0.5%.
  - `TestThresholdFlag`: `cutoff:5000` enables 4999 and disables 5000.
  - `TestGLEntriesAppendOnly`: on embedded Postgres 17 with `payroll.sql` applied, `UPDATE gl_entries` errors.
  - `TestSchemaAppliesOnPG15`: the schema applies cleanly on embedded Postgres 15.
- [ ] **Step 2: Run, expect failure.** Run `go test ./internal/rollout/... ./internal/schema/...`. Expected: FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run, expect pass.** Same command. Expected: PASS.
- [ ] **Step 5: Commit** `feat: payroll schema, routing directory, flag rules`.

### Task 3: `pay` and `attendance` workers with simulated releases

**Files:**
- Create: `internal/attendance/attendance.go`, `internal/pay/pay.go`, `internal/pay/release_2026_10_2.go`, `internal/pay/release_2026_10_3.go`, `internal/pay/release_2026_10_3_fixed.go`, `internal/attendance/release_att_2026_10_1.go`
- Test: `internal/pay/pay_test.go`, `internal/attendance/attendance_test.go`

**Interfaces:**
- Produces:
  - `type DB interface` (subset of `pgx.Tx`);
  - `func pay.RunCycle(ctx context.Context, db DB, dir rollout.Directory, shardID string, tenantID int64, period Period, seq int64) error`;
  - `func attendance.Submit(ctx context.Context, db DB, dir rollout.Directory, shardID string, tc Timecard, seq int64) error`.
- Release behaviour, keyed by `dir.ReleaseOf(shard, service, seq)`:
  - `2026.10.2` and `2026.10.3-fixed`: `hours_accrued = round(sum(timecard hours) × 0.0385, 2)`.
  - `2026.10.3`: the same, except **tenants with `accrual_v2` enabled get `hours_accrued` × 2**. This is the accrual-38 bug.
  - `att-2026.10.1`: a duplicate `timecards` row for 1 in 50 submissions, written on conflict as a second row in `timecards_dup_log` with the same values. This is the `+decoy` fault: a different table and a different symptom.
  - **Slow-write path (all releases):** when the shard's `load` flag is set and the event's seeded coin < 0.5, `RunCycle` writes the accrual in two statements and drops the second half. This is the `+confounded-hidden` mechanism. It is deterministic from `seq`, never from real latency.
  - **Nightly job:** `func pay.NightlyJob(ctx, db, dir, shardID string, tenantID int64, seq int64) error` writes `job_runs`. Under `2026.10.3` it skips (`ran=false`) for tenants in the seeded skip set carried in the directory.
- Writers connect as `payroll_app` with `application_name` = `pay` or `attendance`. The injector never writes these tables directly (see Task 6).

- [ ] **Step 1: Write failing tests** on embedded Postgres with seeded data:
  - `TestAccrualCorrectOnGoodRelease`: hours sum 160 gives accrual `6.16`.
  - `TestAccrualDoubledOnlyForFlaggedTenantsOn2026_10_3`: the flagged tenant gets `12.32`; the unflagged tenant on the same shard gets `6.16`.
  - `TestFixedReleaseIsCorrect`: `2026.10.3-fixed` with the flag on gives `6.16`.
  - `TestPayWritesBalancedGL`: per pay run, `sum(amount) = 0`.
- [ ] **Step 2: Run, expect failure.** Run `go test ./internal/pay/... ./internal/attendance/...`. Expected: FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run, expect pass.** Same command. Expected: PASS.
- [ ] **Step 5: Commit** `feat: pay and attendance workers with simulated releases`.

### Task 4: Seeded traffic schedule

**Files:**
- Create: `internal/traffic/traffic.go`
- Test: `internal/traffic/traffic_test.go`

**Interfaces:**
- Consumes: `baseline.Dataset`, `pay.RunCycle`, `attendance.Submit`.
- Produces:
  - `type Event struct{ Seq int64; Shard string; Kind EventKind; TenantID int64; Payload []byte }`;
  - `func traffic.Schedule(seed uint64, shards []topology.Shard, cycles int, participation map[int64]bool) []Event`. Pure. `participation` nil means every tenant runs a pay cycle. It is used by `+noncompliance`.
  - `func traffic.Apply(ctx, events []Event, open func(shard string) (DB, error)) error`. Events for one shard are applied sequentially in `Seq` order. Each event's DB effect uses `Seq`-derived logical dates, never the wall clock.

- [ ] **Step 1: Write failing tests:**
  - `TestScheduleDeterministic`: two calls are equal.
  - `TestApplyTwiceSameChecksum`: two fresh embedded databases, the same schedule, and `checksum` gives identical results. The checksum is MD5 over `COPY (SELECT … ORDER BY pk) TO STDOUT` for every payroll table.
- [ ] **Step 2: Run, expect failure.** Run `go test ./internal/traffic/...`. Expected: FAIL.
- [ ] **Step 3: Implement.** Put `func checksum.Shard(ctx, db) (string, error)` in `internal/agent/checksum.go`, because Task 8 reuses it.
- [ ] **Step 4: Run, expect pass.** Same command. Expected: PASS.
- [ ] **Step 5: Commit** `feat: deterministic traffic schedule`.

### Task 5: Variant catalogue, spec parser, composition

**Files:**
- Create: `internal/variant/catalogue.go`, `internal/variant/spec.go`
- Test: `internal/variant/spec_test.go`

**Interfaces:**
- Produces:
  - `type Variant struct{ Name string; Base bool; FixKind string /* sql|operational|books */; AvailableFromStep int; RequiresEngine string; Conflicts []string; Apply func(*plan.Builder) error }`;
  - `func variant.Catalogue() []Variant`, which has all 26 rows of the spec table;
  - `func variant.Parse(spec string) ([]Variant, error)`.
- Grammar: `<base>(+<modifier>)*`. Bases: `accrual-38`, `stuck-flag-38`, `gl-38`. Modifiers apply in catalogue order whatever order they're written in, so `accrual-38+unreachable+decoy` ≡ `accrual-38+decoy+unreachable`.
- Conflicts (refused): `+null` with every modifier except `+unreachable` and `+knockoff-calibration`; `+randomised` with `+threshold`; `+randomised` with `+co-rolled`; `+no-overlap` with `+randomised`.
- Unavailable variants produce the error `variant +<name> is available from step <N>`.
- `fleetlab catalogue list` prints name, base or modifier, available step, and a one-line description.

- [ ] **Step 1: Write failing tests:**
  - `TestParseOrderInsensitive`.
  - `TestUnknownVariantRefused`: `accrual-38+nope` errors.
  - `TestConflictingModifiersRefused`: each conflict pair above errors and names both variants.
  - `TestDeferredVariantMessage`: `accrual-38+failover` errors with exactly `variant +failover is available from step 8`.
  - `TestCatalogueComplete`: every spec-table name is present. The list is hard-coded from `PLAN.md` "Variant catalogue".
- [ ] **Step 2: Run, expect failure.** Run `go test ./internal/variant/...`. Expected: FAIL.
- [ ] **Step 3: Implement** with `Apply` stubs. Task 6 fills them in.
- [ ] **Step 4: Run, expect pass.** Same command. Expected: PASS.
- [ ] **Step 5: Commit** `feat: variant catalogue and spec parser`.

### Task 6: Plan builder, truth, and every step-1 variant

**Files:**
- Create: `internal/plan/plan.go`, `internal/plan/builder.go`, `internal/truth/truth.go`, `internal/variant/v_<name>.go` (one per available variant)
- Create: `internal/plan/testdata/golden_hashes.txt`
- Test: `internal/plan/plan_test.go`, `internal/variant/variants_test.go`

**Interfaces:**
- Produces:
  - `type Order struct{ Site topology.Site; Shard string; Kind OrderKind /* set_release|set_flag|set_attr|load|revoke_reader|hotfix|post_gl */; Params json.RawMessage }`;
  - `type Plan struct{ RunID string; Orders []Order; Traffic traffic.Params }`. `RunID` is a random UUIDv4 and is **not** derived from the seed;
  - `func plan.Build(spec string, seed uint64) (Plan, truth.Truth, error)`. Pure, except for `RunID`;
  - `type truth.Truth struct{ RunID, Spec string; Seed uint64; Shards map[string]ShardTruth; Cause map[string]string; Expect Expect }`;
  - `type ShardTruth struct{ Label string /* affected|clean|unknown|decoy */; BadRows map[int64]int; Env string }`;
  - `type Expect struct{ MaxRungWithoutExperiment int; RequiredDesign string; ContainmentRequired, InformativeMissingness, SimpsonFlag bool; LastShard string; RefusedAdjustment []string; HiddenConfounder string; ITT, CACE float64; ExcludedEnvs, TrueAttributes []string; FixKind string }`.
- **Truth is derived, not asserted.** After variants apply orders to an in-memory directory, `Build` computes `Label` and `BadRows` by simulating `traffic.Schedule` against the release and flag state. Labels may never be set by hand, except `unknown` (from `revoke_reader`) and `decoy`.

**Variant constructions.** In each case the test asserts the property in the last column on the built truth or directory, for seeds 1–20:

| Variant | Construction | Asserted property |
|---|---|---|
| `accrual-38` | `accrual_v2` is a hash flag. The release `2026.10.3` wave is 38 shards that have ≥ 1 flagged participating tenant, plus 82 that have none, with ≥ 1 affected shard per site | `count(affected) == 38`; every site has ≥ 1; `Cause == {release:2026.10.3, flag:accrual_v2}` |
| `+decoy` | `att-2026.10.1` on 5 shards outside the affected set | 5 `decoy`; 0 overlap with affected |
| `+unreachable` | `revoke_reader` on one seeded onprem shard | that shard `unknown` |
| `+unknown-informative` | `load` on 70% of affected and 5% of clean shards (a burner query loop held by the agent) | `InformativeMissingness` |
| `+co-rolled` | flag `ui_density` set as a shard attribute on exactly the release-wave shards | `MaxRungWithoutExperiment == 3` (Supported) |
| `+randomised` | the release wave is chosen by hash bucket of the shard ID with a recorded salt | `RequiredDesign == "natural-experiment"` |
| `+simpson` | per-site flag rates and wave sizes are chosen so the raw site-level affected rate is highest on a site other than the one with the highest within-release rate | the raw and stratified orderings differ; `SimpsonFlag` |
| `+bleeding` | `Traffic.Follow = true` (the workers keep cycling after inject) | `ContainmentRequired` |
| `+celebrity` | one affected shard gets a tenant with 10× the employees | `LastShard` = that shard |
| `+null` | the wave gets `2026.10.3-fixed` | 0 affected; `Cause` empty |
| `+no-overlap` | the release goes only to `onprem-a` (PG15) shards, and every one has a flagged tenant | affected == the `onprem-a` set; `RequiredDesign == "clone-a"` |
| `+threshold` | `accrual_v2` uses `rule=threshold` with a seeded cutoff | `RequiredDesign == "rd"` |
| `+noncompliance` | the flag is assigned to 60% of tenants; `participation` is true for a third of them (20% overall) | `ITT`, `CACE` computed from the truth; `CACE ≈ ITT / 0.333` within 1e-9 |
| `+confounded-hidden` | `set_attr san=san-1` (absent from the inventory-visible attributes) on a group; the wave targets that group; `load` there causes the pay worker's two-statement timeout path to drop half an accrual write | `HiddenConfounder == "san"`; `MaxRungWithoutExperiment == 2` |
| `+post-treatment-trap` | under the release, the nightly job skips on a seeded subset; conditioning on `job_ran` flips the sign of the release–defect association | sign flip asserted; `RefusedAdjustment == ["job_ran"]` |
| `+knockoff-calibration` | 200 `shard_attributes` from a Gaussian copula with pairwise ρ drawn from U[0.3, 0.9], independent of the cause | `TrueAttributes == ["release","flag.accrual_v2"]` |
| `+repaired-env` | a `hotfix` order (run by the agent as `payroll_app`, `application_name=pay`) repairs k = 6 affected shards at schedule midpoint | those shards' `Env == "repaired"`; `ExcludedEnvs == ["repaired"]` |
| `stuck-flag-38` | the operational flag `payrun_hold` is stuck on for tenants on 38 shards | `FixKind == "operational"`; 38 affected |
| `gl-38` | release `2026.10.3` posts the accrual liability to the wrong account on 38 shards | `FixKind == "books"`; `BadRows` = wrong entries |

- [ ] **Step 1: Write failing tests:**
  - `TestBuildDeterministic`: for every available spec (each base, plus each single compatible modifier on `accrual-38`, plus `accrual-38+decoy+unreachable`) and seeds 1–20, two `Build` calls give equal `Orders` and `Truth`, ignoring `RunID`.
  - `TestPlanGoldenHashes`: SHA-256 of the canonical JSON of (Orders, Truth minus RunID) for 10 fixed (spec, seed) pairs equals `testdata/golden_hashes.txt`. The file is written on first run with `-update` and committed.
  - One test per table row, asserting the "Asserted property" column.
  - `TestTruthBadRowsMatchSimulatedWrites`: apply the plan's directory and traffic to embedded Postgres for 6 shards (a mix of affected and clean), and count the accrual rows that differ from the correct formula. The count equals `BadRows` exactly.
  - `TestNoInjectorTells`: after the test above, no row in any payroll table has a column value, role or `application_name` that differs between injected and organic writes. Check with `pg_stat_activity` capture and column scans.
- [ ] **Step 2: Run, expect failure.** Run `go test ./internal/plan/... ./internal/variant/...`. Expected: FAIL.
- [ ] **Step 3: Implement** `Builder` (an in-memory directory with `SetRelease`, `SetFlag`, `SetAttr`, `Order(...)`) and each `v_<name>.go`.
- [ ] **Step 4: Run, expect pass.** Same command. Expected: PASS.
- [ ] **Step 5: Commit** `feat: plan builder, derived truth, step-1 variants`.

### Task 7: Sealed truth, vault, scorer

**Files:**
- Create: `internal/truth/seal.go`, `internal/vault/vault.go`, `internal/scorer/scorer.go`, `internal/scorer/ci.go`
- Test: `internal/truth/seal_test.go`, `internal/scorer/scorer_test.go`, `internal/scorer/ci_test.go`, `internal/vault/boundary_test.go`

**Interfaces:**
- Produces:
  - `type Sealed struct{ RunID string; KeyVersion string; WrappedKey, Nonce, Ciphertext []byte }`;
  - `func truth.Seal(t Truth, pub *rsa.PublicKey, keyVersion string) (Sealed, error)`: AES-256-GCM over canonical JSON, with the data key wrapped by RSA-OAEP-SHA256 and `RunID` as AAD;
  - `type vault.Decrypter interface{ AsymmetricDecrypt(ctx, keyVersion string, ciphertext []byte) ([]byte, error) }` (KMS in production, an in-memory RSA key in tests);
  - `func vault.Open(ctx, s truth.Sealed, d Decrypter) (truth.Truth, error)`;
  - `type Verdict struct{ RunID string; Shards map[string]string; Cause map[string]string; Rung int; Flags map[string]bool }` (JSON schema `verdict.v1`);
  - `func scorer.Score(v Verdict, t truth.Truth) Result`, where `Result{ TP, FP, FN, UnknownCorrect int; CauseCorrect bool; Checks map[string]bool /* names only */ }`;
  - `func scorer.Aggregate(rs []Result, byVariant map[string][]int) Report`;
  - `func scorer.ClopperPearson(k, n int, conf float64) (lo, hi float64)`.
- CLI:
  - `fleetlab score --run <id> --verdict v.json`: decrypts in memory and prints only `Result` (counts and check names).
  - `fleetlab score --report runs.txt`: aggregates, with 95% CIs.
  - `--reveal`: Decision 7.

- [ ] **Step 1: Write failing tests:**
  - `TestSealOpenRoundTrip`.
  - `TestSealedTamperFails`: flipping 1 ciphertext byte, or swapping `RunID`, makes `Open` error.
  - `TestSealNeedsNoPrivateKey`: `Seal`'s signature takes only `*rsa.PublicKey`.
  - `TestOnlyScorerImportsVault`: run `go list -deps -f '{{.ImportPath}}' ./...` for each package. Only `internal/scorer`, `internal/vault` and `cmd/fleetlab` may depend on `github.com/3am-si/3am-fleetlab/internal/vault`.
  - `TestScorePerfectVerdict`: TP = 38, FP = 0, FN = 0.
  - `TestScoreUnknownCountedAsUnknownNotClean`: verdict `clean` on a truth-`unknown` shard does not count as correct.
  - `TestClopperPearsonKnownValues`: (0, 200) gives `hi ≈ 0.0182`; (5, 100) gives `[0.0164, 0.1128]` to 4 dp. The rule-of-three line for (0, 200) prints `≤ 1.5%`.
  - `TestScoreOutputHasNoShardIDs`: the printed `Result` does not match `shard-\d{3}`.
  - `TestBurnedRunExcluded`: a run with a reveal event is absent from `Aggregate`.
- [ ] **Step 2: Run, expect failure.** Run `go test ./internal/truth/... ./internal/vault/... ./internal/scorer/...`. Expected: FAIL.
- [ ] **Step 3: Implement.** Use `cloud.google.com/go/kms/apiv1` for the production `Decrypter`.
- [ ] **Step 4: Run, expect pass.** Same command. Expected: PASS.
- [ ] **Step 5: Commit** `feat: sealed truth, vault boundary, scorer with Clopper-Pearson`.

### Task 8: Site agent and the `inject` / `reset` / `traffic` / `checksum` commands

**Files:**
- Create: `internal/agent/agent.go`, `internal/agent/orders.go`, `internal/agent/apply.go`, `cmd/fleetlab/*.go`
- Test: `internal/agent/agent_test.go`, `cmd/fleetlab/inject_test.go`

**Interfaces:**
- Queue layout per site bucket (`gs://<project>-fleetlab-orders-<site>`, `s3://<account>-fleetlab-orders-aws`):
  - `orders/<run_id>/<order_seq>.json` is written by the injector;
  - `acks/<run_id>/<order_seq>.json` is written by the agent;
  - the agent deletes each order after it acks.
- Produces:
  - `func agent.Run(ctx, site topology.Site, bucket *blob.Bucket, open func(shard string) (DB, error), poll time.Duration) error`, which is idempotent per `(run_id, order_seq)` via a local `fleetlab_applied` table on each server;
  - `fleetlab reset --seed N --site <s|all>`: drops and recreates every shard DB from `baseline.Generate` and resets the directory. It is sent as a `reset` order to agents;
  - `fleetlab inject <spec> --seed N`: builds the plan, seals the truth with the public key from KMS `GetPublicKey`, writes `runs/<run_id>.sealed` to `gs://<project>-fleetlab-truth`, enqueues orders and waits for acks. **Stdout is exactly `run_id=<uuid>`**;
  - `fleetlab traffic --run <id> --batch | --follow`;
  - `fleetlab checksum --site all`: prints `shard-NNN <md5>`.

- [ ] **Step 1: Write failing tests** using `fileblob` buckets and embedded Postgres playing three "sites" (2 shards each):
  - `TestAgentAppliesOnlyOwnSite`.
  - `TestAgentIdempotentOnRedelivery`: the same order delivered twice applies once.
  - `TestInjectStdoutOnlyRunID`: stdout matches `^run_id=[0-9a-f-]{36}\n$`.
  - `TestRunRecordHasNoSeedOrShards`: the run record and the order JSON don't contain the seed's decimal form, and the run record has no `shard-` string.
  - `TestEndToEndDeterministicLocal`: (reset 1, inject `accrual-38+decoy` seed 5, traffic batch, checksum) twice gives identical checksums.
- [ ] **Step 2: Run, expect failure.** Run `go test ./internal/agent/... ./cmd/...`. Expected: FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run, expect pass.** Same command. Expected: PASS.
- [ ] **Step 5: Commit** `feat: site agent and inject/reset/traffic/checksum commands`.

### Task 9: Terraform, GCP lab (Cloud SQL, on-prem VPC, truth vault)

**Files:**
- Create: `terraform/gcp/{main.tf,sql.tf,onprem.tf,vault.tf,iam.tf,variables.tf,outputs.tf}`
- Test: `terraform/gcp/tests/gcp.tftest.hcl`

**Interfaces:**
- Consumes: Step 0 `lab-gcp` remote state (`cluster_name`, `registry`).
- Resources:
  - Cloud SQL PG17 `gcp-sql-a` and `gcp-sql-b`, private IP, with replicas toggled by `var.replicas_enabled`;
  - VPC `onprem` with **no `google_compute_firewall` of direction INGRESS except source = the VPC CIDR on 5432**, plus Cloud NAT;
  - 4 VMs (`onprem-a`, `onprem-a-replica` PG15; `onprem-b`, `onprem-b-replica` PG17), `e2-small`, Container-Optimized OS, attached SA `fleetlab-agent-onprem`, no external IP;
  - buckets `fleetlab-truth` (uniform access, versioning, 365 d retention lock **off** in the lab) and `fleetlab-orders-{gcp,onprem}`;
  - KMS key `truth-seal` (`ASYMMETRIC_DECRYPT`, `RSA_DECRYPT_OAEP_4096_SHA256`).
- IAM (the vault boundary):
  - `cloudkms.cryptoKeyDecrypter` on `truth-seal` goes **only** to GSA `fleetlab-scorer`;
  - `cloudkms.publicKeyViewer` goes to `fleetlab-injector`;
  - `storage.objectCreator` (not viewer) on `fleetlab-truth` goes to `fleetlab-injector`;
  - `storage.objectViewer` on `fleetlab-truth` goes only to `fleetlab-scorer`;
  - each orders bucket is readable only by its site's agent SA and the injector.
- Outputs: `truth_key_version`, `truth_bucket`, `orders_buckets`, `sql_private_ips`, `onprem_internal_ips`.

- [ ] **Step 1: Write a failing `terraform test`** with a mocked provider. Assert:
  - exactly one IAM member binding grants the `cloudkms.cryptoKeyDecrypter` role, and its member is `serviceAccount:fleetlab-scorer@…`;
  - no ingress firewall on `onprem` has a source range outside the VPC CIDR;
  - no onprem VM has an `access_config` (external IP);
  - `onprem-a` image metadata selects PG 15;
  - no `google_service_account_key` resource exists.
- [ ] **Step 2: Run, expect failure.** Run `terraform -chdir=terraform/gcp test`. Expected: FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run, expect pass.** Same command, plus `terraform validate`. Expected: PASS.
- [ ] **Step 5: Apply (founder approval first; costs money).** Commit `feat(gcp): fleetlab Cloud SQL, on-prem VPC, truth vault`.

### Task 10: Terraform, AWS lab

**Files:**
- Create: `terraform/aws/{main.tf,rds.tf,iam.tf,variables.tf,outputs.tf}`
- Test: `terraform/aws/tests/aws.tftest.hcl`

**Interfaces:**
- Consumes: Step 0 `lab-aws` remote state (VPC, `cluster_name`).
- Resources:
  - RDS PG17 `aws-rds-a`, private subnets, with a read replica toggled by `var.replicas_enabled`;
  - S3 `fleetlab-orders-aws` (block public access, SSE-S3);
  - IAM role `fleetlab-agent-aws` with a Pod Identity association for namespace `fleetlab`, SA `site-agent`, scoped to that bucket;
  - a Secrets Manager–managed master password (`manage_master_user_password = true`), with the agent reading only the `payroll_app` and admin secret ARNs.
- Outputs: `rds_endpoint`, `orders_bucket`, `agent_role_arn`.

- [ ] **Step 1: Write a failing `terraform test`.** Assert:
  - RDS has `publicly_accessible = false`;
  - the bucket has all four public-access blocks set to true;
  - the agent role policy resources are only the orders bucket ARN, `/*` under it, and the two secret ARNs;
  - no `aws_iam_access_key` resource exists.
- [ ] **Step 2: Run, expect failure.** Run `terraform -chdir=terraform/aws test`. Expected: FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run, expect pass.** Same command. Expected: PASS.
- [ ] **Step 5: Apply (founder approval first).** Commit `feat(aws): fleetlab RDS and agent identity`.

### Task 11: `lab up|down|status`, idle stopping, cost record

**Files:**
- Create: `cmd/fleetlab/lab.go`, `terraform/aws/idle.tf`, `terraform/gcp/idle.tf`, `COST.md`
- Test: `cmd/fleetlab/lab_test.go`, plus additions to both tftest files

**Interfaces:**
- `fleetlab lab down`:
  - sets `replicas_enabled=false` (replicas can't be stopped);
  - Cloud SQL `activation_policy=NEVER`;
  - RDS `StopDBInstance`;
  - stops the onprem VMs;
  - scales the `fleetlab` deployments in GKE/EKS to 0.
- `fleetlab lab up` reverses this and blocks until `status` reports all 5 servers plus replicas ready.
- `fleetlab lab status` prints one row per server: state, replica state, and the last state change.
- Idle guards:
  - an EventBridge Scheduler rule runs `rds:StopDBInstance` on `aws-rds-a` daily at 03:00 UTC, unless the tag `fleetlab:keep-running=true` is set (Review Focus 5);
  - a Cloud Scheduler job sets `activation_policy=NEVER` nightly under the same tag rule.

- [ ] **Step 1: Write failing tests:**
  - `TestLabDownPlanOrder`: with a fake cloud client, `down` deletes the replicas *before* stopping their primaries. RDS refuses to stop an instance that has a read replica.
  - The tftests assert that the scheduler rule exists and targets `aws-rds-a`.
- [ ] **Step 2: Run, expect failure.** Run `go test ./cmd/... && terraform -chdir=terraform/aws test`. Expected: FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run, expect pass.** Same command. Expected: PASS.
- [ ] **Step 5: Write `COST.md`.** Give hourly and monthly cost for the states **up**, **down** (idle) and **destroyed**. Break it down per line item: Cloud SQL ×2, RDS, 4 VMs, NAT ×2, the GKE/EKS cluster fees from Step 0, and storage. Take the values from the GCP and AWS pricing pages for the chosen regions, and cite the date. After 24 h in `down`, add the measured cost from the billing exports next to the estimate.
- [ ] **Step 6: Commit** `feat: lab up/down, idle guards, cost record`.

### Task 12: Deploy and step-1 acceptance

**Files:**
- Create: `deploy/helm/values-gcp.yaml`, `deploy/helm/values-aws.yaml`, `deploy/onprem/cloud-init.yaml`, `scripts/e2e-step1.sh`, `cmd/fleetlab/verifylab.go`, `.github/workflows/{ci,release}.yml` (copy the Step 0 release workflow, image `fleetlab`)

**Interfaces:**
- `fleetlab verify-lab --run <id>` runs as the scorer identity. It opens the truth, counts the actual bad rows per (shard, tenant) through the agents' read path, and prints `MATCH` or the number of mismatches, never the values.
- `e2e-step1.sh` steps, in order:
  1. `lab up`;
  2. for seed in 1 2: `reset --seed 1`, `inject accrual-38+decoy+unreachable --seed $seed`, `traffic --batch`, and `checksum > run$seed.txt`;
  3. redo seed 1 and diff its checksum against `run1.txt`;
  4. assert seed 1 ≠ seed 2;
  5. `verify-lab` on each run;
  6. the vault negative check;
  7. `lab down`.

- [ ] **Step 1: Cut `v0.1.0`.** The release must produce a signed `fleetlab` image. Check: `cosign verify` passes, as in Step 0 Task 8.
- [ ] **Step 2: Deploy the site agents and workers** to GKE (`fleetlab` namespace, KSA bound by WI), EKS (Pod Identity) and both onprem VMs (cloud-init pulling the image by digest). Check: each agent logs `site=<s> polling`.
- [ ] **Step 3: Run the acceptance check.** Run `scripts/e2e-step1.sh`. Expected lines:
  - `DETERMINISTIC: seed 1 rerun identical (257/257 shards)`
  - `SEED-SENSITIVE: seed 1 vs 2 differ`
  - `TRUTH MATCHES LAB: run1 MATCH, run2 MATCH`
  - `VAULT: injector decrypt → PERMISSION_DENIED; injector read truth object → 403; scorer decrypt → ok`
  - `IDLE: all servers stopped`
  - `STEP 1 ACCEPTED`
- [ ] **Step 4: Confirm `COST.md`** has its measured row filled in (Task 11 Step 5).
- [ ] **Step 5: Commit** `feat: step-1 acceptance`. Paste the Step 3 output into the PR body as evidence.
