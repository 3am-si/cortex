# How these plans are made

This is the checklist used to write the M1 step plans in this folder (`2026-10-18-m1-step0-foundation.md`,
`2026-10-18-m1-step1-fleetlab.md`). Use it for steps 2–13. It records the reasoning, not only the steps. A future
writer should then make the same kinds of calls the same way, or change them on purpose.

## 0. Ground rules

- `PLAN.md` is the spec. A step plan never contradicts it silently. Where the plan must differ (an impossibility, a
  conflict, a gap), the difference goes under **"Decisions made in this plan (not in the spec; founder to check)"**,
  with the reason.
- Write **one plan per M1 build step** (the table under "Milestone 1 → Build order"). `PLAN.md` spans 14 steps and
  12 repos. One plan for all of it would be unreviewable, and could not be executed task by task. Each step
  plan must end in something testable on its own: that step's acceptance criteria.
- The plans follow the `superpowers:writing-plans` skill: the header, Global Constraints, Review Focus, then tasks
  in TDD form (failing test → run → implement → run → commit). Load that skill before writing.
- Write plans in plain, full sentences. Caveman-style compression applies to chat only, never to files.

## 1. Gather the spec for the step

- [ ] Find the step's row in `PLAN.md` "Milestone 1 → Build order". Copy its **Acceptance** text verbatim into
      Global Constraints. Everything in the plan works backwards from that sentence.
- [ ] `grep -n` `PLAN.md` for every noun in the step row (e.g. for step 1: `fleetlab`, `vault`, `scorer`,
      `injector`, `seed`). Read each hit's section. Relevant rules are spread across Architecture, Invariants,
      Reliability, Learning and trust, Verification, Production gates and Open questions.
- [ ] List the step's dependencies on earlier steps' outputs. Name the exact Terraform outputs, Go packages,
      env vars and flags, so the plan can say "Consumes: …".
- [ ] Check which **Open questions** (`PLAN.md` "Open questions") block or shape the step. Never answer one
      yourself. Make a reversible default, and list the question for the founder.
- [ ] Check the local toolchain (`command -v go buf terraform …`) and existing repos (`gh repo list 3am-si`).
      Missing tools and repos become a "Before you start" section, not silent assumptions.

## 2. Decide scope inside the step

- [ ] Ask: what is the smallest build that satisfies the acceptance sentence exactly? Leave out anything that
      belongs to a later step's acceptance (YAGNI). Examples:
      - Step 0 has one RPC (`GetVersion`), not resource APIs.
      - Authorization is an allowlist, because Cedar is step 7.
      - Cross-cloud calls wait for the relay (step 3).
- [ ] If the spec asks for something that needs a component from a later step, **catalogue it now and defer it**,
      with an explicit "available from step N" error. Example: step 1's seven deferred variants.
- [ ] Look for contradictions between spec sections and surface them; don't paper over them. Examples found so
      far:
      - `+no-overlap` needs a PG15 server holding exactly the affected shards.
      - "Services on GKE" can't reach an on-prem VPC with no inbound rules.
      - `+sqlserver-skip` needs SQL Server in an all-Postgres M1 lab.

## 3. Make the security and honesty calls explicitly

These are where a plausible-looking plan goes wrong. Each one needs a test that pins it.

- [ ] **No static keys.** Identity comes only from Workload Identity, Pod Identity, GitHub OIDC, or a VM-attached
      SA. Every Terraform task gets a tftest asserting that no key, access-key or IAM-user resource exists.
- [ ] **Fail closed.** No token, an empty allowlist or an unparsable input is a refusal, never "allow all". Pin
      it with a test.
- [ ] **Anything that fetches a caller-supplied URL** (e.g. the AWS STS verifier) validates host, scheme and
      action *before* any network call. The test asserts 0 requests were made.
- [ ] **Public side channels.** Keyless cosign writes repo and workflow names to the public Rekor log, so the
      plans use a KMS key with `--tlog-upload=false`. Ask "what does this publish?" for every external service.
- [ ] **Evaluation honesty (ESL discipline):**
      - truth is *derived* from the state the lab actually has, never asserted;
      - truth is sealed so only the scorer can open it, enforced at three levels: IAM, a Go import-boundary test,
        and a live PERMISSION_DENIED check;
      - the seed is secret, because determinism plus a known seed reveals the truth;
      - injected data has no tells;
      - a revealed run is burned and excluded from evaluation;
      - every rate is reported with a Clopper–Pearson interval.
- [ ] **Determinism** means a pure function plus a committed golden-hash file. Use `math/rand/v2` PCG, which is a
      specified algorithm, so the same seed gives the same output across machines and Go versions. Never call
      `time.Now()` in generated data.
- [ ] **Idle cost.** Everything billable has a `down` state and a guard against the cloud undoing it (RDS
      auto-restarts after 7 days, and won't stop while it has a read replica).

## 4. Lock the structure before writing tasks

- [ ] Write a file-structure table: one responsibility per file or package, split by responsibility, not by layer.
- [ ] Give each task an **Interfaces** block with exact names, signatures, env vars, CLI flags, output formats,
      and exact values from the spec (e.g. `2026.10.3`, `accrual_v2`, `257`, shard ranges). An implementer
      sees only their own task, so this block is the contract between tasks.
- [ ] Size tasks as the smallest unit that a reviewer could reject while approving its neighbour. Fold setup
      and scaffolding into the task that needs them. Security-critical pieces get their own task, e.g. the
      AWS verifier is separate from the Google one.
- [ ] Order tasks so pure logic (testable locally, with embedded Postgres and no Docker or cloud) comes first,
      and cloud apply and acceptance come last. Cloud applies cost money, so each one is marked
      "founder approval first".

## 5. Write the tests that carry the decisions

- [ ] A test step gives the test name and its assertions with the spec's exact values. A code step gives the
      signature, file and pinned values, and the implementer writes the body. Include a code block only for an
      algorithm the signature and tests don't determine.
- [ ] Write the **Review Focus** last-but-one: the five inputs or failure modes the spec implies but no task's
      happy path exercises, most likely first. Then add each one's test to the task that owns that code.
- [ ] The final task is always an **acceptance script** that prints one line per acceptance clause and ends
      with `STEP N ACCEPTED`. Its output is pasted into the PR as evidence.

## 6. Self-review before handing over

- [ ] **Coverage:** every clause of the step's acceptance and every spec rule found in §1 maps to a task.
- [ ] **Names:** names and types match across tasks, e.g. `NewGoogleVerifier`, `truth.Truth` and `checksum.Shard`
      are spelled the same everywhere. Every table, column or function a task uses is defined by some task.
      Step 1's first draft missed `timecards_dup_log`, `job_runs`, the slow-write path and the nightly job.
- [ ] **Counts:** check every count against the spec, e.g. the variant table has 26 rows, not 27.
- [ ] **Proportion:** the plan is decisions, not a transcript of the code. If code blocks dominate, replace them
      with signatures and assertions.

## 7. Hand-off

- [ ] Tell the founder in chat:
      - the decisions made (numbered);
      - the open questions only they can answer;
      - the outward or costly actions waiting on approval: repo creation, `terraform apply`, spend.
- [ ] Recommend an execution mode. Subagent-driven is the default for these plans, because the security
      boundaries and the evaluation vault are the kind of thing a per-task reviewer catches and an end-of-branch
      review misses.
- [ ] Save as `docs/superpowers/plans/YYYY-MM-DD-m1-stepN-<topic>.md` and commit only when asked.
