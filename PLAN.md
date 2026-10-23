# 3AM, rebuilt: a polyglot fleet-fix platform inside the customer's cloud

## Context

**The reference incident (a large payroll provider):**
- An engineer at a large payroll provider described what a *normal* incident looks like: 257 sharded database servers, 38 of them
  affected, fixed by an iron-clad SQL script run on each.
- 99.99% of hotfixes are SQL. Only 2–5% of staff may touch a production database.
- After the hotfix: Jira → code fix → next-day release. The provider runs mostly on GCP, with the rest on-prem; banks look the same.

**Today's 3AM can't do this:**
- It is ~33k lines of single-process Python with file-based state and a single-writer ledger.
- One SQL connector is one database. `checks.pick_target` refuses ambiguity, and nothing fans out across shards.
- Drift re-reads every database every 10 min. There is no AWS or GCP integration.
- Its *safety model* is right: proof by live checks, declared actions, System 1 veto, policy and quorum, rehearsal,
  verification and a hash-chained ledger. That model carries over as specification.

**How this plan was built:** it was rebuilt by treating five books as the authority:
- *System Design Interview Vol. 1* (Xu): sizing, queues, IDs, idempotent delivery.
- *Designing Data-Intensive Applications* (Kleppmann): consensus, fencing, isolation, replication, event logs,
  integrity.
- *Elements of Causal Inference* (Peters, Janzing, Schölkopf): what counts as proof of a cause.
- *Elements of Statistical Learning* (Hastie, Tibshirani, Friedman): how not to fool ourselves with fleet statistics.
- *A First Course in Causal Inference* (Ding): design-based proof, i.e. randomization tests, negative controls,
  sensitivity analysis, overlap, natural experiments.

A second review checked the plan against 24 primary papers: logs and transparency, consensus and leases, clocks,
testing, knockoffs and invariance, CDC, messaging, protection and LLM-agent security. Each change it caused cites
the paper, and the paper index is at the end after the book index.

Every decision below traces to a chapter or a paper; the indexes are at the end. "Ironclad" here means three things:
- every safety property has a mechanism and a test;
- every causal claim has a stated estimand, identifying assumptions and a falsification check;
- every remaining risk is named, in the failure-mode and residual-risk tables near the end.

## Founder decisions (binding)

- **Scale:** petabytes. Never ingest; push queries to the data.
- **Reach:** go all the way. Scope, prove before a DBA is paged, draft, rehearse, two-person approval, waves, verify,
  then Jira plus a code-fix PR.
- **Fail-proof first:** the platform must be fail-proof before it touches a database.
- **Repositories:** multi-repo and polyglot, each language where it is strongest:
  - Rust for the safety kernel and runner;
  - Go for the control plane and cloud;
  - Java for JDBC drivers and CDC (see the note under Architecture);
  - Python for models and statistics;
  - TypeScript for the console.
- **Deployment:** inside the customer's GCP *and* AWS at once, plus on-prem through outbound-only relays.
- **Console:** works like the AWS/GCP console; functionality over design.
- **Positioning:** "Own the write path". Milestone 1 is the 38-of-257 incident.

## Competitive position: Traversal (the Goliath)

**Who:**
- ~$53M from Sequoia, Kleiner Perkins and Amex Ventures.
- Customers: Amex, DigitalOcean, PepsiCo, a Fortune 100 financial-services firm and a crypto exchange.
- They ship about two features a week.

**How it works:**
- An edge Processor (logs only, clustered ~1000:1, shipped to their cloud) and an outbound Connector.
- An auto-built Production World Model.
- A Causal Search Engine that runs thousands of parallel tests. Confidence = "data completeness"; self-reported
  RCA accuracy is 75–82%.
- Surfaces: Slack/Teams Workers, alert triage, Knowledge Bank, MCP, API.

**Gaps:**
- No database integration.
- Recommendations, not actions.
- BYOC on AWS only, vendor-operated over a VPN. No on-prem or air-gapped option.
- Logs leave the customer.
- SOC 2 only, with no tamper-evident trail.
- Their "causal" engine is observational search. By the ECI standard (below) it can reach *Correlated* or
  *Supported*, never *Proven*.

**Our wedge:**
- Fleet-scale, *interventionally proven*, rehearsed, two-person-approved SQL fixes.
- In regulated, hybrid and sharded estates, on GCP, AWS and on-prem/air-gapped.
- Operated by the customer.
- "Traversal tells you what probably broke; 3AM proves it and fixes it."

**To match:**
- Slack/Teams Workers, an MCP server and a sessions API.
- A knowledge bank; alert triage; post-mortem drafts.
- Zero per-service configuration.
- Integration parity: CloudWatch, Azure, Dynatrace, AppDynamics, Elastic, Loki/Tempo, Sentry, Confluence,
  incident.io.
- A published "proven vs guessed" benchmark with confidence intervals.

**Option:** be the governed actuator behind any root-cause tool, theirs included, via MCP.

## The books boundary (founder constraint, absolute)

A bank will never let an AI touch its books, not even at 100% accuracy. **3AM never holds a key to the books.**

Each customer declares its book tables: GL, journals, balances, transactions, payroll lines and customer financial
records. For those, sqlguard refuses every write and no write credential is ever issued.

| Class | Who executes | 3AM's output |
|---|---|---|
| **Books** | **A human DBA, with their own key** | A signed **change package**: proven scope, the script, rehearsal on a clone, the expected row-by-row effect per shard, the inverse, the verification query. 3AM verifies afterwards and ledgers it. |
| **Operational state** | 3AM, after approval | Locks, sessions, read-only mode, replication, failover, stuck batch flags, pools, restarts, scale, config. |
| **Non-book data fixes** | Opt-in per customer and class, after a shadow trust record | Same pipeline as operational state. |

**How the DBA runs a book package (their choice):**
- **(a) A plain SQL package** for their own tooling and change process: sqlcmd/psql, Flyway/Liquibase, or attached to
  a ServiceNow change.
- **(b) The signed deterministic `3am-run` under the DBA's own identity and credentials.** It contains no AI. It
  enforces sqlguard, waves, before-images, the fence, the premise re-check, the `_threeam.applied` marker and the
  journal, so 38 shards is one command with every safety rule.

**Book fixes default to compensating entries** (DDIA ch. 11; Helland 2015, "accountants don't use erasers"). That
means a reversal row plus a correcting row, with a reason code and the incident reference. They are never an
in-place `UPDATE` when the schema is ledger-style; that is how the bank's own accountants would do it.

**Either way**, 3AM verifies all 38 shards afterwards, and the ledger records who executed what, with which key.

## The invariants (compiled into keel, runner, sqlguard and vet; each has a TLA+ property or a test)

**Safety, which must hold under any fault** (DDIA ch. 8 system model: partially synchronous, crash-recovery):

1. **NoWriteToBooks.** No write credential for a declared book table ever exists.
2. **TwoPersonRule.** Execution requires two distinct approvals of the same `plan_hash`. Neither approver may be the
   author or the requester, both need WebAuthn step-up, and approvals expire. Any drift invalidates them.
3. **OnlyProvenActs.** An executable plan needs a **Proven** verdict on the proof ladder (below). A Supported verdict
   may draft and rehearse, never execute.
4. **ContainBeforeRepair.** A data repair needs evidence that the producing change is contained (flag off, release
   rolled back, job paused). The alternative is an explicitly approved "repair while bleeding" mode, which re-sweeps
   and reports rows written after the fix separately.
5. **PremiseInTxn.** Inside the fix transaction, the proof predicate is re-run with `FOR UPDATE`, and its count and
   checksum must equal the approved values. Any difference aborts the transaction and halts the plan. This is
   application-level SSI.
6. **FenceInDatabase.** The fix transaction runs
   `UPDATE _threeam.fence SET token=$t WHERE shard=$s AND lane=$lane AND token<$t`. Zero rows aborts. The token is
   keel's etcd revision, so a paused runner with a stale lease cannot commit. This is Chubby's sequencer, checked by
   the resource itself (Burrows 2006).
   - **One fence row per lane.** `ops` is for 3AM's operational and data fixes; `books` is for human-run change
     packages. A book package's long run window is then never invalidated by an unrelated operational fix on the
     same shard. The lanes write disjoint tables (book tables versus non-book tables), so they never race on rows.
   - **Fence tokens stay monotonic across a restore or DR failover.** A restored etcd starts with its revision
     bumped past the highest fence any shard holds (`etcdutl snapshot restore --bump-revision`); see HA and DR.
   - **Targets that can't check a fence** (cloud action cards, on-prem tools, anything outside SQL) get a
     **lock-delay** instead. After a lease is lost uncleanly, no runner may act on that target until the old
     lease's **recorded deadline** plus clock uncertainty has passed. The deadline is stored in the lease's value;
     the store lease's own TTL is not a bound, because etcd refreshes lease TTLs when its leader changes.
7. **AtMostOnceApply.** The `_threeam.applied(op_id, step_id, plan_hash, fence, before_image_sha)` row commits in the
   same transaction as the fix. Reconciliation reads the marker; it never re-runs blindly.
8. **NoCrossShardAtomicity.** No 2PC or XA, ever. Each shard commits on its own. The plan guarantees that every shard
   ends verified-fixed, verified-untouched, or halted with known state.
9. **TimeoutIsUnknown.** Every RPC timeout is "outcome unknown". Only the marker resolves it. A timeout cannot tell
   "committed, ack lost" from "slow" from "dead" (Fischer, Lynch & Paterson 1985). So no safety property may
   depend on a timeout firing in time; timeouts serve liveness only.
10. **UnknownIsNotClean.** An unreachable or late shard is Unknown. It is never imputed, and proof must hold under
    every completion of the Unknowns (ESL ch. 9.6: unreachability is not missing-at-random).
11. **HaltOnFailover.** A DB failover or timeline change on any shard halts the plan. Committed shards are then
    re-verified, because a marker that is gone means the fix is gone.
12. **NoDualWrites.** keel appends; everything else follows from its log through an outbox. Derived state (Postgres
    projections, cubes, the global index, the topology graph) is never consulted for a safety decision. Every
    authority read (approved? lease live? halt set?) is an etcd **linearizable** read, never `serializable`
    (Ongaro & Ousterhout 2014).
13. **DeterministicScripts.** sqlguard refuses `now()`, `random()`, `nextval` and `LIMIT` without a total
    `ORDER BY`. Every trigger, rule, cascade and CDC subscriber on a target table must be declared.
14. **NamedHuman.** Every write has a named, accountable human approver in the ledger. A model is never the
    accountable party. Shared approver accounts are refused: the principal is the unit of accountability
    (Saltzer & Schroeder 1975).
15. **NoStaleReports.** Every report carries the keel epoch, lease and fence it was produced under. Examples are a
    probe result, a runner ack and a site's late events. A report from a superseded epoch is quarantined and never
    merged into state. This is the Azure vNext bug class, where a node already declared dead re-added replicas
    (Deligiannis et al. 2016).
16. **ReasonerHasNoAuthority.** No LLM output is ever authority. The reasoner cannot grant a rung, approve, choose an
    assignment, hold a credential or reach the network. Everything it proposes passes deterministic gates and two
    humans. See "The LLM boundary".
17. **IndependentPreApplyCheck.** No shard write gets a credential unless `vet`, a second, diverse implementation,
    agrees with the runner. `vet` independently checks the plan, approvals, fence, scope and expected after-image
    (Avizienis 1985; Saltzer & Schroeder's single-fault invulnerability).

**Liveness:** a plan eventually finishes or halts, and an approval eventually expires. Both are stated under partial
synchrony and explicit fairness, because FLP rules out termination in a fully asynchronous model.

**Verification is of the cause, not the symptom** (ECI). A plan that only treats a symptom is labelled "mitigation,
not repair" and can never close an incident.

## Architecture: a narrow verified write path, a broad asynchronous read surface

**Systems of record (DDIA Part III):**
- The customer databases are the truth about data.
- **etcd, via keel,** is the truth about plans and authority.
- The **transparency-log ledger** is the truth about what happened.

**Everything else is derived**, rebuildable by replaying the log, and versioned as immutable builds that switch
atomically (DDIA ch. 10).

**What is coordinated:**
- Only two things run synchronously and linearizably: **authority** (approve, lease, fence, halt) and **customer-DB
  writes**.
- Everything else is idempotent asynchronous dataflow: scope, evidence, projections, notifications, Jira and learning
  (DDIA ch. 12).
- That keeps the coordinated surface small enough to model completely in TLA+.

The control plane never touches a database. Every cloud VPC or account and every on-prem DC is a **Site**. A
**runner** at each site does the work locally.

| Service | Lang | Role |
|---|---|---|
| `keel` | Rust | Safety kernel. Event-sourced: commands are validated by compare-and-set on **etcd** (3 voters, 5 for banks, one region, 3 zones) and appended as immutable events. One active leader through an etcd lease. Holds plans, approvals, intents/outcomes, leases, fences (= etcd revision), halt, budgets, autonomy, the JIT credential broker and Cedar authorization. **We don't write our own Raft** (DDIA ch. 9). Every command carries its op ID inside the same etcd transaction, so a client retry can't apply it twice. Every write is a CAS on the leader key's revision, so an ex-leader's write fails in etcd itself. Every response carries the leader epoch (Raft §8; Chubby). FoundationDB is the alternative, decided by the M1 step-2 spike. |
| `runner` | Rust | Site agent: outbound-only. Guarded read probes, routed to replicas with the **observed position** recorded. Journaled write steps with fence, premise re-check and marker. Encrypted before-images, uploaded by **pre-signed single-object URL**. Signs evidence with a per-site key. **Subscribes** to policy, pack, routing and fence-config streams and keeps local versioned copies. Has a **jeopardy** state: when its conservative local lease runs out, it starts no new write step and finishes or aborts the step in flight before the real lease expires (Chubby). Rejects messages from an older keel epoch. Plaintext customer data never touches its disk: keys sit in locked memory, swap is off, and scratch is encrypted. |
| `sqlguard` | Rust (lib, CLI, WASM) | Fail-closed AST rules, identical at approval, at execution, in `3am-run` and in the console. A parse failure or an unknown statement type is a refusal. |
| `vet` | Go | **The diverse pre-apply checker** (invariant 17). It runs at each site as its own process, with a different owner from keel and runner. Before the broker issues a shard credential, `vet` checks the following itself: the two approvals, via its own linearizable etcd read through a read-only gateway (etcd's client port is never exposed); the fence; scope ⊆ approved keys; statement hashes; the expected after-image hash against the ledgered rehearsal. Then it signs a per-step token. Right after `COMMIT` it reads the marker and after-image checksum back from the shard, and it halts the next wave on any mismatch. It reads raw sources only, never keel's projections or APIs. |
| `control` | Go | API (ConnectRPC, Google AIPs), inventory and topology graph (in memory, one process), intake, scope, planner, notify, follow-through. Stateless replicas. |
| `cloud` | Go | GCP and AWS read adapters generated from Discovery and Smithy; curated write action cards; change-stream ingesters. |
| `infer` | Python | **Deterministic statistics and causal engine, with no LLM and no network.** Computes the detection residuals, the stratified differential, permutation null, model-X knockoff+ selection, ICP, stability selection and conformal calibration. It produces proof-ladder rungs 1–3 only. Fixed seeds, pinned versions, replayable. |
| `reasoner` | Python | LLM proposals: hypotheses, SQL and code-fix drafts. Templated read-only tools and verbatim values. No network egress except the model endpoint, and no memory across incidents. Temperature 0, with prompts and seeds recorded, and its *output* ledgered, because temperature 0 is not replayable. **Proposes; never grants a rung** (invariant 16). |
| `auditor` | Go | An independent read-only consumer, deliberately *not* in keel's language (DDIA ch. 8/12). **It detects after the fact; it doesn't tolerate faults at execution time. `vet` does that** (Avizienis 1985). It re-verifies past fixes on a schedule. It cross-checks ledger ↔ markers ↔ keel outcomes from raw sources, using its own Merkle verifier. It rebuilds projections and diffs them, reconciles the relay count audits, and runs quarterly restore drills. When it stalls, it reports "audit missing", never "audit clean". |
| `console` | TypeScript | AWS/GCP-console style. Every view is stamped "as of keel revision N". |
| CDC | runner (Postgres); packaged Debezium Server (SQL Server, MySQL) | Postgres logical decoding (pgoutput) is consumed by the runner itself: one process per database server multiplexes that server's per-database slots, because Debezium's Postgres connector serves one database per process, and 64 shards on a server would mean 64 JVMs. SQL Server CDC and the MySQL binlog go through packaged Debezium Server, whose connectors cover many databases each. All under the CDC contract below. Java runs where it is strongest, as packaged Debezium and as a JDBC driver sidecar for Oracle/DB2 in M2. We don't maintain a first-party Java service. |

**Required core for an install** (DDIA ch. 12, the fewest moving parts):
- **Kubernetes, etcd, Postgres 16, an S3-compatible store with Object Lock, NATS JetStream.**
- **ClickHouse is an optional large-estate tier.** The default is Postgres with time-partitioned facts and
  materialised cubes.
- OpenBao only where the customer has no Vault or CyberArk.
- Every component must justify itself against "could Postgres or etcd do this?"

**Protocols:**
- Protobuf, buf and ConnectRPC.
- `buf breaking` in CI on every repo; field tags are never reused.
- Unknown fields are preserved on re-publish.
- Safety fields live in a `critical` extension that old readers must *reject*.
- Each ledger record embeds its schema descriptor hash (DDIA ch. 4).
- Rolling upgrades go servers first; a runner refuses any plan whose schema it can't read.

**Bus:**
- JetStream **streams** (ordered, retained, partitioned by shard) for changes, evidence and ledger events.
- JetStream **work queues** for notifications and follow-through.
- Leaf nodes over outbound 443 are the relay transport.
- Retained streams make **backtesting** possible: replay last quarter through a new pack or model and score it
  offline before it runs live.
- **Bus contract** (Kreps et al. 2011):
  - Delivery is at-least-once, so every consumer is idempotent, keyed by event ID or op ID.
  - Order holds only within a partition (a shard). Cross-shard reasoning orders by keel revision or log position
    carried in the event, never by arrival.
  - Each stream has explicit retention, sized for "replay last quarter". An alarm fires before any consumer's lag
    reaches the retention edge, because lag past retention is silent loss.
  - Publishes are R3 and acked. keel's outbox waits for the ack.
  - Each runner publishes per-window event counts per stream; `auditor` reconciles them with what arrived. That
    proves end-to-end that nothing was lost across the relay.
  - Every message carries its schema descriptor hash. A consumer refuses a schema it doesn't know.

**Ledger** (Crosby & Wallach 2009; RFC 6962; Chuat et al. 2015):
- A **transparency log**: a Merkle tree over records, with inclusion and consistency proofs.
  - It uses **RFC 6962's construction exactly**: SHA-256, leaf/node domain separation (`0x00`/`0x01`), tree shape and
    proof algorithms. Off-the-shelf CT verifiers and the customer's own tool can then check it.
  - Verification depends only on record bytes, never on understanding the record type.
- Each head is sealed via Cloud KMS, AWS KMS or an HSM, and **co-signed by a customer-held witness key**, so 3AM can't
  rewrite history.
  - Signing is batched per head, not per record.
  - Each head records its key version. Key history is itself ledgered, and a verifier accepts any key valid at the
    head's time. So rotation never breaks old proofs.
- **Receipts with a maximum merge delay (MMD).** Every party that hands keel a record gets a signed promise of
  inclusion within the MMD: an approver, a runner reporting a commit, or a DBA running a book package. `auditor`
  and the customer can demand an inclusion proof for every receipt.
- **A fresh signed head at least every MMD**, even when idle. A missing head is an alert.
- **Gossip, so 3AM can't show different histories to different parties** (a split view).
  - Every party that sees heads keeps the largest verified head: console, runners, `3am-run`, `auditor`, `vet` and
    the witness.
  - Parties exchange (head, newer head, consistency proof) in existing RPC metadata. No new service.
- **The witness enforces as well as signs.** It stores every head it signs and refuses to co-sign one that isn't
  consistent with the last.
  - Two conflicting signed heads are self-proving. They open an incident against 3AM itself, raised to the
    customer.
  - A head stale beyond the MMD is a warning, which needs confirmation before it is reported.
- **Merkle aggregation** over shard, incident and class (books vs operational) makes "every write to shard X" a
  provably complete query. That is the question a bank auditor asks.
- The legacy hash chain and its Ed25519 segments still verify inside it.

**IDs and time:**
- UUIDv7 everywhere.
- Ordering within a plan uses the etcd revision, never wall clocks.
- Every runner event carries three timestamps: occurred-at and sent-at on the site clock, received-at on the control
  clock. Site clock offset is estimated continuously.
- **Clock uncertainty ε is measured, never assumed** (Corbett et al. 2012, TrueTime).
  - Each runner polls several sources: the cloud time service, the control-plane clock and on-prem NTP.
  - It rejects outliers Marzullo-style, and ε grows linearly with time since the last sync.
  - Events carry ε, and `infer` widens their intervals.
- **Leap-second smearing differs by source.** GCP and AWS smear; plain NTP steps. Around a leap second they can
  disagree by ~0.5 s, beyond the gate. Each site records its time-source type, and a leap-second window counts as
  high uncertainty.
- **Precedence is defined as `C.latest < O.earliest`.** Overlapping intervals mean "order uncertain".
- Within a shard, ordering uses log positions (LSN, GTID), not DB clock timestamps.
- Leases use monotonic durations measured locally.
- **Every safety deadline is a value, checked against a bounded clock.** Approval expiry, cooling-off and lease
  deadlines are stored in the record and checked against keel's clock with its uncertainty: a deadline holds only
  while `now + ε < deadline`, and a delay has passed only when `now − ε ≥ eligible_from`. The keel leader stamps
  `now` into each command's event, so `apply` stays a pure function of its inputs and replays exactly. A keel node
  whose clock offset exceeds the gate cannot lead. Store leases only garbage-collect: etcd refreshes every lease's
  TTL when its own leader changes, so a store lease's lifetime is not a bound.
- The keel lease length is chosen from the halt-latency requirement. A hard leader loss pauses waves for up to one
  lease, which is safe because of the fence, and visible in the console.
- A runner more than ±250 ms off, or with ε above the gate, is refused write steps.

**End-to-end operation ID** (DDIA ch. 12):
- Minted at the approver's click or CLI command.
- Carried console → keel event → runner step → marker row → ledger → Jira → PR.
- Retries collapse to one effect.

**Data is hazardous** (DDIA ch. 12):
- Probes return summaries (counts, capped IDs, hashes), never raw rows.
- Before-images are encrypted per incident under a data key wrapped by the customer's KMS. At retention end the key is
  destroyed (**crypto-shredding**), so the ledger keeps the hashes and the content is gone.
- **Why crypto-shredding is the only real deletion** (Gutmann 1996):
  - Overwriting in place is unreliable on SSDs and cloud object stores. Wear levelling, remapped blocks, replicas,
    snapshots and backups all keep old copies.
  - So the rule covers **every place plaintext could land**: runner scratch and temp files; clone and rehearsal
    databases, on customer-KMS volumes and destroyed by key or volume deletion, never by `DROP`; core dumps; swap.
  - Data keys live in locked memory, are short-lived and are zeroised after use.
  - The KMS key-destruction event is **ledgered**, so deletion is provable, not promised.
- NATS streams, Postgres projections and logs never carry raw row values.
- No PII goes to any model outside the customer boundary.

**CDC contract** (Andreakis & Papapanagiotou 2020; Andreakis 2026, Generalized DBLog):
- **No CDC operation may take a table lock or global read lock on a customer DB.** No CDC component runs a
  locking snapshot: Debezium runs with `snapshot.mode=no_data`, and chunk reads are done by the runner.
- **The source must provide:**
  - a commit-ordered log;
  - positions that denote log prefixes;
  - stable row identity;
  - full row images;
  - complete chunk reads;
  - enough retention.
- **Per engine:**
  - **Postgres.** A commit record can reach the WAL before the transaction is visible to new snapshots, so a chunk
    read can miss it and a deleted row can be **resurrected**. Chunk reads therefore compare their own snapshot
    (`pg_current_snapshot()`) and widen the window until it covers everything before the low watermark. Tables
    whose events feed evidence use `REPLICA IDENTITY FULL`.
  - **SQL Server.** Chunk reads run under SNAPSHOT/RCSI. Locking READ COMMITTED can skip or double-read rows.
  - **MySQL.** GTID mode and `binlog_order_commits` on; multithreaded replicas preserve commit order.
- **Chunk reads run on the primary**, or on a replica only after its replay position is confirmed past the low
  watermark.
- **Postgres chunk reads use read-only LSN brackets** (M1 step 4 Decision 1). Inside one `REPEATABLE READ`
  transaction on the primary, `pg_current_wal_lsn()` read before and after the chunk bounds the window, and
  `pg_current_snapshot()` drives the widening. No 3AM write is needed. A **watermark table** (a 3AM-owned write
  alongside `_threeam.fence` and `_threeam.applied`, open question 3) stays the fallback for an engine without
  readable log positions; MySQL's read-only GTID variant covers estates where writes are refused.
- **A slot and WAL budget per server.** Postgres slots are per database, so a server with 64 shards holds 64 slots,
  and every walsender reads the server's whole WAL.
  - Each server has an aggregate WAL budget (default 20 GB across all its slots); `max_slot_wal_keep_size` is the
    budget divided by the slot count, and at least 1 GB.
  - When retained WAL passes 80% of the budget, the runner drops the lowest-priority slot and invalidates its units,
    rather than let the customer's disk fill.
  - The decoding overhead (WAL read amplification and primary CPU) is measured per server and reported. A server
    over its overhead budget gets fewer slots, never silent lag.
- **A failover with a discontinuous position history** invalidates CDC-derived evidence for that shard until its
  units are rebuilt. This extends invariant 11: fail closed and rebuild.
- **Replay into a populated sink** clears the repaired scope or uses tombstones. Projections rebuild into a fresh
  table and swap.
- **Polling `updated_at > t` is outside the contract.** It can't see deletes, and its stamps aren't commit-ordered.
- **Per-table change rates from CDC are free Detect signals**, for example the insert rate dropping after a deploy.
  They cost no query against the customer DB.

**The LLM boundary** (Greshake et al. 2023; Ruan et al. 2024; Saltzer & Schroeder 1975 on confinement):
- **Everything the reasoner reads may be attacker-written**: DB rows, log lines, error text, tickets, commit
  messages, column comments, runbooks. Retrieved text acts like code. Prompt instructions and filters are not
  controls: even the best safety-prompted agent failed 23.9% of ToolEmu's cases.
- **Structural controls:**
  - **No egress.** No URL fetch, web search or email, which blocks exfiltration by side effect, multi-stage payloads
    and remote control.
  - **No memory across incidents**, so an injection can't persist. Learned knowledge enters only through
    human-reviewed packs.
  - **Templated read tools only**, with bound parameters. The reasoner never composes free queries, so its reads
    can't become a covert channel.
  - **Verbatim values, enforced mechanically.** Every literal in a draft must byte-match an evidence ID, or the draft
    is refused.
  - **Ambiguity fails.** A target key that resolves to more than one row, tenant or payee fails the scope probe.
    Drafts never pick with `LIMIT 1` or "the first match".
  - **Hidden-text stripping.** Zero-width characters, homoglyphs, Base64 blobs and HTML/Markdown comments are
    stripped or flagged before evidence reaches the model, and what was stripped is logged.
  - **Budgets.** Each call has a time and token budget. When it runs out, the system records "reasoner unavailable"
    and falls back to pack templates, so injection can't stall an incident.
- **LLM judges are advisory only.** ToolEmu's evaluator agreed with humans at κ ≈ 0.48 with 73% recall: good for
  finding failures in testing, far too weak to gate anything.
- **LM-emulated environments may red-team the reasoner, but never produce evidence.** Emulated outcomes are not
  observations. madsim and fleetlab remain the only test truth.

## Repositories

| Repo | Purpose |
|---|---|
| `3am-proto` | APIs, events and ledger schemas; generated clients for Go, Rust, TypeScript and Python |
| `3am-safety` | Rust workspace: keel, runner, sqlguard, `3am-run`, ledger (Merkle log + legacy verifier), broker, sim (madsim), spec/ (TLA+), hermitage/ (isolation conformance) |
| `3am-control` | Go: control plane + `auditor` (separate binary, separate review owners) |
| `3am-vet` | Go: the diverse pre-apply checker. Its own repo, with an owner who doesn't write keel or runner. Written from the TLA+ invariants, not from keel's code |
| `3am-cloud` | Go cloud layer (apigen, action cards, change ingesters) |
| `3am-research` | Python: `infer` (deterministic stats) and `reasoner` (LLM), packaged as separate images |
| `3am-packs` | Signed `pack.v2` packs: declarative probes, scope rules, mediator probes, fix templates, verify probes |
| `3am-console` | React 19 + TanStack |
| `3am-cli` | Go, gcloud-style, composable over stdin/stdout |
| `terraform-provider-threeam` | Terraform provider |
| `3am-deploy` | Helm, Terraform, Marketplace listings, relay appliance, packaged Debezium Server (SQL Server, MySQL), cosign + SBOM |
| `3am-fleetlab` | The 257-shard lab, the variant catalogue, the injector, and the scorer with a vaulted ground truth |
| Existing | `warden`, `pinata` (single-estate regression) |

`3am` becomes **3am-classic**, frozen except for strangler adapters.

**Strangler sequence:**
1. Classic publishes its ledger events to NATS.
2. keel becomes the only gate. The `sql.py` rules become runner invariants. `policy.py` is ported with **zero
   divergence on golden replay of past ledgers**, checked with a paired McNemar test on every decision (ESL ch. 10).
3. Go takes over intake, orchestration, inventory and checks, with Pinata run on both engines and the results diffed.
4. Python stays for `infer` and `reasoner`.
5. The new console replaces the old one; `query.ts` and `search.py` map to AIP-160.

**The `pack.json` → `pack.v2` converter:**
- Keeps classes, checks, `expect`, bound remediations, risk and sealing.
- Adds `scope` probes, a `mediator` probe, `fix_templates` and `verify` probes.
- `fix_templates` carry typed SQL, the tenant key, a self-guard, the inverse and a compensating-entry form for books.
- Probes and rules are declarative (typed SQL plus Datalog-style predicates), never code, so sqlguard can analyse them
  and the planner can push them down (DDIA ch. 2).

## The fleet fix pipeline

Each stage's output is a content-addressed artefact you can open, diff and re-feed; the CLI composes them (DDIA ch.
10). The DAG runs in keel's scheduler. Errors are typed: retryable ones back off with jitter; terminal ones halt
(Xu ch. 14).

### 0. Detect

**Baselines** (ESL ch. 5, 6, 9):
- Each steady-state signal gets an additive seasonal baseline, S(t) + T(t) + ε: natural splines or local-linear fits,
  which stay unbiased at the right edge ("now"), where moving averages lag.
- Bootstrap confidence bands; Huber loss so outliers don't drag the baseline.

**Fleet effects are removed first** (ECI ch. 8, half-sibling regression):
- Each shard's anomaly is its **residual after regressing on the other shards' same metric**.
- That strips out shared payroll-cycle load and backup windows without hand-tuning.

**Drift** (ESL ch. 14.2.4):
- A classifier is trained on "current window vs reference window". An AUC near 0.5 means no drift.
- Its feature importances localise what changed, as a pointer for a human, not as proof.

**Alerts:**
- A loss matrix sets the thresholds; a missed payroll fault and a false page are priced explicitly (ESL ch. 2).
- Every threshold is chosen on held-out incidents, never on the ones it is scored against.
- **Conformal intervals on the residuals** (Shafer & Vovk 2008). Split-conformal calibration on held-out windows
  gives each alert a stated false-alarm rate, with no distributional assumption beyond exchangeability.
  - Windows with seasonality and deploys aren't exchangeable. Calibration is therefore **Mondrian**: per
    payroll-cycle phase and per incident class.
  - It is redone after every release. An exchangeability test runs on the calibration set; when it fails, the
    alert says "uncalibrated".
  - Low **credibility** (the input is unlike every past window) feeds the reject option.
  - Wording is about the procedure: "this detector is wrong ≤ 5% of the time", never "95% chance this shard is
    affected".

### 1. Scope (under 1 min, the whole population, always)

**Fan-out:**
- The pack's scope probe fans out to **all 257 shards**, never just the ones that alerted. Conditioning on alerts or
  tickets manufactures correlation (ECI, Berkson).
- Probes go to the replica by default. Each returns a summary plus the **observed position** (replay LSN, AG LSN or
  GTID set).
- A fleet scope is a **vector of per-shard positions**, never "a point in time" (DDIA ch. 5).
- **Each per-shard claim is coordinate-bound.** The probe runs in one statement snapshot whose position is
  recorded in the same transaction: Postgres `pg_current_snapshot()` plus replay LSN; MySQL the GTID set. Without
  that, "clean at position P" has no defined coordinate (Generalized DBLog).

**Limits:**
- 5 s statement timeout, 100 ms lock timeout, a row cap, and a scan budget proven by `EXPLAIN`: the estimated rows
  read and the plan cost stay under a per-probe cap. An aggregate over a whole table is allowed when it fits the
  budget; the statement timeout stays the hard stop.
- At most k probes in flight per DB server, with a priority queue in front so incidents beat steady state (Xu ch. 9).
- Each shard has its own deadline. A shard past its deadline is **Unknown**. Late results merge in as ledgered
  corrections only if their epoch is current (invariant 15).
  Latency is stored as HDR histograms, which are added, never averaged (DDIA ch. 1).

**Output:** Affected / Clean / Unknown. Unknowns are tested for being *informative*: does unreachability itself
correlate with the fault? If so, that is flagged (ESL ch. 9.6).

### 2. Describe

**The affected set as a rule:**
- PRIM bump-hunting or a small rule ensemble describes the affected set as an interpretable conjunctive rule, e.g.
  `release=2026.10.3 ∧ flag.accrual_v2 ∧ schema≥57` (ESL ch. 9, 14, 16).
- The rule's precision and recall over the 257 are reported.
- The rule becomes the Datalog scope predicate and later the standing detector.

**Guards:**
- High-cardinality attributes (server ID, shard ID) are excluded from rule search, because they overfit by
  construction (ESL ch. 9.2.4).
- A reference built from the product of marginals lets rare-but-strong associations surface ("vodka ⇒ caviar",
  ESL ch. 14.2).

### 3. Prove: the proof ladder (replaces "all three hold")

A verdict is the highest rung reached. **Only Proven may lead to an executable plan.** The reasoner proposes
candidates for every rung but grants none.

**Identification before estimation** (Ding ch. 2, 10, 18). Every verdict starts with a written analysis record:
- **The estimand.** By default this is the effect *on the affected units*: "what did release R do to the shards
  and tenants it reached" (ATT).
- **The unit.** Shard, tenant or row-batch. The same shard before and after a change counts as two units, so a
  before/after comparison alone never proves anything.
- **A well-defined intervention.** Release R build B, flag F = on, config key K = v. "High load" is not a cause
  we can act on: no causation without manipulation.
- **The identifying assumptions**, chosen from a fixed list: randomization, ignorability given the backdoor set,
  overlap, exclusion restriction, continuity at a cutoff.
- **The analysis plan.** The adjustment set and test statistic are fixed in the ledger *before* any outcome probe
  runs (design stays blind to outcomes). Re-running with different specifications until one "works" is
  specification search, and it is refused. Alternatives are only ever shown together, as a full specification
  curve.

Where the assumptions identify only a range, the verdict reports **bounds**, not a point. That covers Unknown
shards and missing outcomes (Manski bounds with Imbens–Manski intervals).

| Rung | Evidence | Verdict | Granted by |
|---|---|---|---|
| 5 | **Intervention.** A designed clone experiment (spec in Rehearse) whose exact Fisher randomization test gives p ≤ 0.01, with the test achieving that bound at least once *before* it runs. Or a **natural experiment**: a randomised rollout (tenant-hash bucket) analysed as ITT/CACE, or a threshold rollout analysed by regression discontinuity. | **Proven** | runner (clones), `infer` (FRT, IV, RD) |
| 4 | **Mechanism (front-door).** A row-level mediator probe ties the bad rows to the writer: release/build ID, `job_run_id`, or CDC source metadata with its LSN. Plus temporal precedence, evaluated with **clock-uncertainty intervals**: overlap means "order uncertain", which is not proof. Audit columns (`updated_at`, `modified_by`) alone count as Supported at most. They can't see deletes and aren't commit-ordered, so they need corroboration from the log (Generalized DBLog). | **Proven**, with a "relative to observed variables" caveat | runner probes |
| 3 | **Invariance.** ICP across sites, clouds, DCs and time windows. The selected set is **contained in** the true parents with probability ≥ 1−α, relative to observed variables, with no hidden confounding and no selection. It does *not* say every parent was found (Peters, Bühlmann & Meinshausen 2016). **Plus** an adjusted differential that passes the statistical honesty rules below. | **Supported**: draft and rehearse only | `infer` |
| 2 | **Adjusted association.** A differential stratified by the topology's backdoor set (site, cloud, server, engine version, tenant-size band, shard age), never adjusting for descendants of the suspect. Raw and adjusted effects are shown side by side, and a sign flip (Simpson's paradox) is flagged. | **Correlated**: the investigation continues and the DBA sees the gaps | `infer` |
| 1 | Raw association or temporal coincidence; LLM hypotheses; Granger-style lag tests | **Hypothesis** | reasoner |

**Statistical honesty rules for `infer`** (ESL ch. 3, 7, 8, 11, 18):
- **Charge for the search.** The differential tests hundreds of attributes, so the effective degrees of freedom far
  exceed the number of features in the final rule. Significance comes from a **permutation null**: shuffle
  affected/clean labels *within strata*, then compute plug-in FDR and q-values. Local fdr is reported as the
  posterior probability that an attribute is null, which feeds the Bayesian engine.
  - BH is only guaranteed under independence or PRDS, and correlated attribute tests satisfy neither. So the
    *selection* uses knockoff+ (below), with Benjamini–Yekutieli as the stated fallback. BH q-values are shown as
    descriptive only.
  - Asymptotic logistic/GLM p-values are never reported once p/n ≳ 0.1. With 257 shards and 30+ attributes that is
    always the case: about 17% of null p-values fall below 0.05 (Candès et al. 2018).
- **Decoys are model-X knockoffs, never permuted copies** (Barber & Candès 2015; Candès et al. 2018).
  - A column-permuted copy keeps its own distribution but loses its correlation with the other attributes. So a
    real correlated null beats it easily: 45.6% FDR at a nominal 20% in their simulation. A decoy check built that
    way gives false reassurance.
  - Knockoffs are therefore *sampled* from a model of the attribute distribution, which the inventory history
    provides. Each knockoff is exchangeable with its original given all the others: second-order (mean and
    covariance) knockoffs, with approximate SDP on a block-diagonal covariance.
  - **Selection uses the knockoff+ threshold**, which gives exact FDR under arbitrary correlation. Plain knockoff
    overshoots in the 1–3-discovery regime RCA lives in.
  - If a knockoff still ranks as a candidate cause, the run is capped at Correlated (the NIPS-2003 "artificial
    probes" lesson).
  - Knockoffs are random draws. They are redrawn R times, and each candidate's selection frequency is reported.
  - **Co-rolled attributes use group knockoffs**: the selection unit is the group {release R, flag F}.
  - **Where an attribute was randomised** (tenant-hash rollout), its conditional distribution is known exactly. The
    conditional randomization test for it *is* the FRT below: same idea from two directions.
  - Model-X assumes i.i.d. rows. It is used only within strata where shards are exchangeable given the topology.
    Design-based claims over the finite 257 stay with the FRT.
- **Discover on one half, confirm on the other.** Sample splitting over shards or time windows; selection happens
  only on the discovery half. Standard errors after model search are invalid otherwise.
- **Stability selection with stated error control** (Meinshausen & Bühlmann 2010; Shah & Samworth).
  - The whole pipeline, screening included, runs on **complementary pairs** of disjoint halves.
  - The resampling unit is the **whole interference group and time block**, never individual shard-windows. Shards
    that share a server are dependent, and windows are autocorrelated, so smaller units overstate stability.
  - q and the threshold are chosen so the bound E[falsely selected] ≤ q²/((2π_thr − 1)p) is ≤ 1 (or a stated α).
    The bound is reported.
  - Frequencies are computed for **groups** of correlated attributes too. Near-duplicates split their votes and can
    both fall below the threshold. The verdict then says "{release R, flag F}: stable as a group; separation needs
    rung 4/5".
  - Votes are not probabilities, so they are calibrated before display.
- **Correlated suspects can't be separated by statistics.** When release and flag were co-rolled on the same shards
  (the irrepresentable condition), lasso, ICP and trees cannot tell them apart. The verdict says so, and only rung 4
  or 5 can resolve it. Grouped or elastic-net penalties keep correlated attributes together rather than picking one
  at random.
- **Interpretation tools are not causal.** Variable importance, partial dependence and SHAP-like scores are shown
  as "where to look", never as evidence. A conditional expectation can show an effect for a variable the outcome
  doesn't depend on (ESL Ex. 10.12).
- **Bet on sparsity.** RCA assumes a sparse mechanism shift (ECI ch. 2), and the estimators match it: L1 and nearest
  shrunken centroids (ESL ch. 16, 18). If the shift isn't sparse, the verdict is "not identified".
- **Abstain.** Every classifier has a reject option priced by the loss matrix. "Don't know" routes to a human.
  System 1 vetoes from the same Bayes rule with asymmetric losses (ESL ch. 2, 4).

**Design-based rules for `infer`** (Ding):
- **Exact tests first.** Each p-value comes from a Fisher randomization test (FRT) using the actual assignment
  mechanism, with permutations within strata or pairs.
  - The test statistic is *studentized* (robust t, or Lin-adjusted t). That makes it exact under the sharp null and
    still valid under the weak null.
  - Monte-Carlo p-values use (1 + #extreme)/(1 + R), so they stay valid for any R.
  - The console shows the randomization distribution next to the p-value.
- **Report the smallest achievable p-value.** A design that can never reach the threshold is refused before it
  runs. For example, 3 + 3 units can't go below 0.05, and 5 matched pairs reach 1/32.
- **Overlap gate.** If the affected shards' attribute combination has no clean counterparts (propensity ≈ 1),
  observational comparison is impossible.
  - The verdict says "no overlap: experiment required" and stays at Correlated or below.
  - Trimming to the overlap region is allowed only with the changed estimand stated.
- **Choose covariates carefully.** Adjust only for pre-change confounders and outcome predictors.
  - Never adjust for post-change variables: "shards whose job ran", "shards that alerted", replica lag after the
    deploy. Doing so is collider, M- or Z-bias, or conditioning on a principal stratum.
  - Rollout bucket and wave IDs are *instruments*: they are used as IVs, never adjusted for.
- **Negative controls are mandatory below rung 4.** Each one that fails caps the verdict one rung lower.
  - *Negative-control outcome:* a table or metric the suspect cannot touch shows no "effect".
  - *Negative-control exposure:* a sibling change shipped to similar shards at a similar time, which cannot cause
    the defect, shows no "effect".
  - *Placebo time:* the pre-change window shows no "effect" (a lagged-outcome test).
- **Sensitivity analysis on every observational claim.** Report the E-value, or Rosenbaum's Γ* for matched
  shard pairs: how strong a hidden confounder would need to be to explain the finding away.
  - Supported needs Γ* ≥ 2 (configurable per estate).
  - In big fleets, sampling p-values shrink but confounding doesn't, so sensitivity, not the p-value, is the
    headline number.
- **Natural experiments.**
  - *Rollout bucket vs code path:* "flag assigned" and "code path actually ran" differ (noncompliance). The
    analysis reports ITT, plus CACE = ITT / compliance.
  - *Weak compliance:* the verdict rests on the ITT test and Fieller–Anderson–Rubin intervals, never on the Wald
    ratio. The IV inequalities are checked as a falsification test.
  - *Threshold rollouts and deploy instants:* a tenant-ID cutoff, a wave boundary or a deploy timestamp is analysed
    as regression discontinuity (sharp or fuzzy). This uses local-linear fits, a bandwidth sensitivity sweep and a
    McCrary density check for manipulation. In time, the clock-uncertainty interval is the minimum bandwidth.
  - *Several independent rollouts:* combine them by Egger-style regression, so one invalid instrument shows up as
    a non-zero intercept.
- **Estimators.** Matching with bias correction, or AIPW with Hajek normalisation and stated truncation. Never raw
  Horvitz–Thompson. Doubly robust estimators are *doubly fragile* when both models are wrong, so their result never
  overrides a failed negative control.
- **Mechanism claims are controlled direct effects** (Ding ch. 26–28).
  - "Release → code path → bad rows" is shown by *forcing* the code path on or off in clones: cde(m), which can be
    falsified.
  - We never report "x% of the effect went through M" (a natural indirect effect). That rests on cross-world
    assumptions no experiment can check.
- **Finite population.** The 257 shards *are* the population, so fleet inference is design-based over that
  finite population. Superpopulation (IID) inference is reserved for cross-estate learning. A lab result has
  internal validity only: it transfers to a customer only after that estate's own shadow record.

**ICP rules for `infer`** (Peters, Bühlmann & Meinshausen 2016, and its discussion):
- **Environments are exogenous**, never descendants of the outcome. Sites, clouds, DCs and calendar windows qualify.
  "Shards that alerted", "reachable shards", "shards with a ticket" and "shards a DBA already patched" are refused,
  the same Berkson ban as in Scope.
- **Informative Unknowns cap rung 3.** If unreachability correlates with the fault, ICP on the reachable shards
  conditions on a selection variable. That is the case where ICP turns anti-conservative, so the verdict is
  capped, not just flagged.
- **Interventions on the outcome** happen: a DBA hotfix, an earlier partial repair, a regional rollback. Run ICP on
  **pairs of environments** and take the multiplicity-adjusted union. Exclude environments the ledger shows were
  repaired.
- Method II (residual invariance) is the default. Lasso pre-screening, |S| ≤ 3, and the screening step are recorded
  in the analysis record, because pre-screening costs α (coverage becomes 1−2α).
- **"Every set rejected" is a signal**: no invariant explanation among the observed attributes. It feeds the
  missing-topology detector as "hidden cause or intervention on the outcome likely".
- With one environment, ICP claims nothing: the empty set is always invariant.

**Interference and hidden structure:**
- **SUTVA** fails for shards that share a server, replica set or storage. Those form an **interference group**, which
  is the unit for canaries and for transfer of results (ECI ch. 6.9).
- **Missing topology.** When symptoms co-vary with a low rank the declared topology doesn't explain, a latent shared
  component (SAN, switch, hypervisor, pooler, KMS key) is proposed as a candidate node.
  - The null is the singular values of column-permuted data, i.e. a gap-statistic style test (ESL ch. 14).
  - Graphical-lasso edges supply undirected candidates (ESL ch. 17).
  - A human confirms the node; direction needs an intervention.
- **Every claim carries its assumptions:** "relative to observed variables {…}". Below rung 4 it uses PAG wording:
  "X → Y, or a hidden common cause of both" (ECI ch. 9).

### 4. Contain

- The plan's first step targets the **cause's producer**: flag off, rollback, or job pause, each an operational
  action card.
- Repair waits until the mediator probe shows **no new bad rows** for a declared window (invariant 4).

### 5. Draft

**Bindings:**
- A parameterised script with per-shard bindings taken verbatim from probe output.
- The source is a pack template, or the reasoner with values verbatim. The verbatim rule is checked mechanically:
  each literal must byte-match an evidence ID (see "The LLM boundary").
- Book tables get compensating-entry templates.

**Build, then swap** (DDIA ch. 10):
- Rehearsal computes the exact **after-image per row**: keys, before and after values, checksums.
- Execution applies only those keyed changes.
- A row whose before-image no longer matches is skipped and reported, never "fixed differently".

### 6. sqlguard (fail closed)

- DML on approved tables only.
- A conjunctive tenant predicate on the declared key, with no widening `OR`.
- A self-guarding WHERE, so a re-run touches 0 rows.
- Determinism (invariant 13) and declared side effects.
- An indexed access path, proven by `EXPLAIN`.
- Single atomic `UPDATE … SET col=f(col) WHERE <proof> AND <guard>`; never read-modify-write in the runner.
- An exact expected rowcount per shard plus a hard cap.
- PK-range batches sized under ~1 s. Each fix transaction has a time and row budget, and larger scopes are chunked
  with a marker per chunk. A long `FOR UPDATE` premise re-check holds locks on a payroll DB (FoundationDB's 5 s
  rule).
- A derived `SELECT … FOR UPDATE` before-image.
- An inverse guarded on "current value = after-value".
- **Per-engine isolation:**
  - Postgres runs at `REPEATABLE READ`; a serialization failure is retried.
  - MySQL requires locking reads.
  - SQL Server uses SNAPSHOT or UPDLOCK.
  - The runner refuses any engine/version that hasn't passed the **Hermitage-style conformance suite** in fleetlab
    (DDIA ch. 7).

### 7. Rehearse (a causal experiment, not only a dry run)

**Methods:**
- Cloud SQL/AlloyDB clones or Aurora fast clones, one per server.
- Otherwise a rolled-back transaction with strict timeouts.

**What it must show:**
- Counts equal expected.
- The post-check passes.
- **A second run affects 0 rows.**
- **The inverse restores the checksum.**
- Lock wait and duration stay within budget.
- **The clone experiments from rung 5** (spec below).

The fix's predicted effect is a falsifiable prediction. If rehearsal or verification fails it, **the causal model is
falsified** and the verdict drops to Unproven.

**Clone-experiment specification** ("block what you can, randomize what you cannot", Ding ch. 3–8):
- **Design A: tenant-randomised within a clone.**
  - Clone an affected server. Inside it, randomly assign the suspect intervention per tenant: flag on/off, or
    route through release R vs R−1.
  - Run the job or traffic replay, then count bad rows per tenant.
  - Analysis: a completely randomised experiment with many units and an exact FRT. It is cheap: one clone.
  - SUTVA within a clone is a stated assumption, checked by comparing bad-row rates of control tenants against the
    unmanipulated baseline clone.
- **Design B: matched pairs across servers.**
  - Take two clones of the same snapshot per server. A coin decides which one gets do(cause) (on a clean server) or
    do(¬cause) (on an affected server).
  - Use ≥7 pairs, so the exact sign test reaches 1/128 < 0.01. Pre-period metrics enter as covariates (Lin
    estimator or gain score).
- **Design C: reproduction plus placebo.**
  - When the software is deterministic, a single do(cause) clone that reproduces the exact row pattern is strong
    mechanism evidence.
  - It counts toward rung 5 only alongside a placebo clone (a no-op change through the same deploy path) that does
    *not* reproduce it.
- **Fixed in advance:** the design, the randomisation seed, the assignment and the test statistic are ledgered
  before the experiment runs. The runner randomises; the reasoner never chooses the assignment.
- **Each clone and its results are an immutable, ledgered DataSet**, so "which clone proved this" stays answerable
  (Helland 2015). A clone is destroyed by key or volume deletion.

Clone cost is budgeted per incident, and Design A is preferred.

### 8. Approve

- The plan is immutable (`plan_hash`) and approved under the two-person rule (invariant 2).
- Each approval records the **hash of the exact view the approver saw**: scope table, rehearsal diff, evidence IDs,
  policy version, the reasoner output, and whether it was vetoed (DDIA ch. 12, reads are events).
- **The approval view puts deterministic evidence first.**
  - The scope table and rehearsal diff are rendered above any LLM text and are never summarised by it.
  - Reasoner prose is labelled untrusted.
  - LLMs write persuasive, well-cited, wrong summaries (Greshake et al. 2023). Two humans must not rubber-stamp a
    narrative.
- The scope is re-probed at start, and drift invalidates the approval.
- **Versioned inputs** are pinned, and a later change to any of them is drift: the routing-directory version (DDIA
  ch. 6), flag config, schema, pack and policy (DDIA ch. 11, time-dependent joins).
  - Every safety decision references a **version ID, never a mutable name** like "current pack" or "latest policy"
    (Helland 2015).
- **Prescripts** (Saltzer & Schroeder 1975):
  - Approval expiry is one.
  - Add a **cooling-off delay** on edits to declared book tables, change packages and approver groups.
  - Add a **customer-designated "court order" principal**, whose signal is required to unlock break-glass.
- **Complete mediation.** Cedar decisions are re-evaluated at execution. Any cache of them has a short, stated TTL.

### 9. Execute in waves

**Wave order:**
- Canary **interference group** → 5 → 12 → the rest, with bake times and health gates.
- **Canary choice is randomised, not hand-picked.** Within interference groups, the canary is drawn by
  rerandomization, so canary and holdout are balanced on covariates (Mahalanobis threshold).
- **The health gate is an FRT.** Canary vs still-untouched shards, with pre-period metrics as covariates, using a
  studentized statistic that compares over the bake window. That makes "canary looked fine" a test with a known
  false-pass rate, not eyeballing.
- The highest-revenue "celebrity" shard is never in the canary or an early wave, and gets its own bake (Xu ch. 1).
- Waves avoid the customer's declared payroll windows.

**Per-shard protocol:**
1. keel commits `INTENT(op_id, plan, shard, step, attempt, fence)`.
2. The runner checks it is on the writable primary and records server identity and timeline.
3. `vet` checks the step independently and signs a step token (invariant 17). The runner exchanges its capability
   plus that token for a JIT credential. The broker refuses unless it gets both.
4. `BEGIN` → timeouts → fence update (invariant 6) → premise re-check (5) → encrypted before-image → statement →
   assert rowcount → post-check → marker (7) → `COMMIT`.
5. keel records `OUTCOME`. `vet` reads the marker and checksum back from the shard; a mismatch halts the next wave.
6. If a runner's session with keel dies, every later step of that plan on that runner fails. So the plan ends as
   "steps 1..k done, rest not started", never with gaps (Chubby's suffix-only loss).

**Halts:**
- Halt triggers: rowcount mismatch, canary failure, failed verification, health regression, replication lag, drift,
  failover (11), the window ending, lease loss or **jeopardy**, a **Degraded** runner, a `vet` refusal or mismatch,
  or a human halt (console, CLI or Slack).
- A **Degraded** runner is one whose latency falls far outside its own phi-accrual history: a gray failure whose
  waves pause and whose shards turn Unknown (DDIA ch. 8).
- Committed shards are never auto-reverted unless that was pre-approved. Otherwise the inverse is a new plan that
  needs a fresh two-person approval.

### 10. Verify

**Checks:**
- The **cause's** proof probe is false on all 38, read on the primary or on a replica whose position is at least the
  fix's commit position.
- Symptoms are cleared. A symptom-only plan can't close the incident.
- **Recovery is shown against concurrent controls.** "The symptom went away after the fix" is a before/after
  comparison of different units. Verification uses difference-in-differences against shards untouched in the same
  window, which removes payroll-cycle and time trends.
- **Merkle roots** over the declared tables, bucketed by PK range, prove the 219 untouched shards unchanged. A
  mismatch is localised in O(log n) round trips without moving rows (Xu ch. 6).

**Report:** "38/38 verified, 219 untouched, 0 unknown".

### 11. Follow through

- A Jira defect linking evidence, plan and verification.
- Release diff → candidate code → drafted fix plus a regression test → a PR on CI.
- Notifications and Jira are after-commit effects, keyed by outcome ID (Xu ch. 10).
- The scope rule joins the pack as a standing detector.

### 12. Audit, forever

- The `auditor` re-verifies the fix on a schedule. A later release can silently undo a verified fix, and a regression
  opens an incident.
- keel self-audits continuously: every INTENT has an OUTCOME or a reconciliation, markers equal OUTCOMEs, and the
  ledger verifies. Any discrepancy halts new plans (DDIA ch. 1).

### JIT credentials (operational and opted-in non-book classes only)

- A KMS-signed capability binds the plan hash, wave, shard set, statement hashes and a TTL.
- **GCP:** a time-bound conditional IAM binding on exactly those instances.
- **AWS:** `AssumeRole` with a session policy for `rds-db:connect` on those resource IDs.
- **On-prem:** an OpenBao/Vault dynamic user.
- Credentials never leave the site, and TTLs are the second line of defence after the fence, never the first.
- **This is a capability system issued from an ACL (Cedar)** (Saltzer & Schroeder 1975). So revoking a credential
  already issued is not immediate, and the latency differs by platform:
  - **AWS:** an `AssumeRole` session lasts at least 15 minutes and an RDS auth token is valid for 15 minutes, so
    revocation latency is up to 15 minutes.
  - **GCP:** a conditional IAM binding expires on time, but IAM changes take minutes to propagate, so a grant can
    take effect late and a removal can linger; revocation latency is the binding's expiry plus propagation.
  - **On-prem:** revoking the OpenBao lease drops the dynamic user within seconds.
  - **Everywhere:** none of these ends a database session that is already open.
- So a mid-wave revoke works through halt plus the fence, never through credential expiry. **A halt also terminates
  the write role's open sessions** on every target server (`pg_terminate_backend` by a runner identity that holds
  only `pg_signal_backend`), so an in-flight step is aborted, not left to finish.

## Learning and trust, statistically honest

- **Diagnosis is anticausal** (ECI ch. 5, 8.3). Packs share likelihoods P(symptoms | fault class) across customers
  and re-estimate priors P(fault class) per estate (label shift). This matches the Bayesian engine in
  [[investigation-engine]].
- **Fault classes have several signatures.** Packs model a class as a mixture of prototypes, fitted by EM, matching
  the scenario variants (ESL ch. 6, 12).
- **Combining detectors:** stacking with cross-validated weights constrained to sum to 1. Posterior model
  probabilities over competing hypotheses come from BIC or full Bayes (ESL ch. 7, 8).
- **Evaluation discipline** (ESL ch. 7):
  - The fleetlab ground truth is **vaulted**: the scorer holds it, and no model or pack sees it.
  - Cross-validation is **grouped by scenario family**, never by run (like keeping a speaker on one side of the
    split), and screening happens inside the folds.
  - Results are reported with standard errors and intervals: Clopper–Pearson, or the rule of three, so "0/200
    unsafe" means ≤1.5% at 95%.
  - Simpler models win ties (one-standard-error rule).
  - Two engines are compared pairwise with McNemar.
  - The 4-incident demo set is a release gate, not accuracy evidence.
- **Autonomy changes:**
  - Shadow logs record what 3AM proposed, what the human did, and the propensities.
  - A proposed rule (auto-approve, a new System 1 threshold) is evaluated offline with IPW or doubly-robust
    estimators, with a CI, *before* the quorum vote (ECI ch. 8.2).
  - Remediation is multi-step, and later steps depend on intermediate health, which is time-varying confounding.
    Policies are therefore evaluated with g-methods: sequential IPW under a marginal structural model, or recursive
    g-formula. Never with plug-in outcome models, because of the g-null paradox (Ding ch. 29).
- **Similar past incidents:** retrieval uses metrics invariant to shard identity, time shift and scale, so similarity
  isn't dominated by irrelevant coordinates (ESL ch. 13).
- **Clustering is navigation, never evidence.** Hierarchical clustering imposes structure whether or not it exists
  (ESL ch. 14).

## Cloud and on-prem integration (GCP and AWS together)

**Placement and identity:**
- The control plane runs in the customer's primary cloud. There is a site in every VPC, account and DC.
- **Identity:** no static keys.
  - GKE Workload Identity Federation and EKS Pod Identity.
  - Cross-cloud federation both ways.
  - Org/folder bindings on GCP; Organizations + StackSets on AWS.
- **Reader** (read-only) and **Actor** roles. The Actor role has no standing rights; only the broker grants them.

**Discovery and telemetry:**
- **Discovery:** Cloud Asset Inventory with real-time feeds; Config aggregator and Resource Explorer; Kubernetes
  informers; ServiceNow CMDB and runner discovery on-prem.
- **Telemetry is queried in place:** Cloud Monitoring PromQL, Managed Prometheus/AMP, Cloud Logging, CloudWatch
  Metrics and Logs Insights, Query Insights and Performance Insights, Query Store deltas via the runner.
- Steady state is pre-aggregated under per-project read-quota budgets, which are token buckets (Xu ch. 4).

**Change streams:**
- GCP: Audit Logs → Log Router → Pub/Sub.
- AWS: org CloudTrail + Config + RDS events → EventBridge → SQS.
- GitHub/TeamCity webhooks; Debezium DDL; the **rollout system** (LaunchDarkly, Unleash, internal flags), whose bucket
  assignment becomes a natural experiment.
- All normalised to `changes.v1`, windowed by **event time with per-site watermarks**. A reconnecting site's
  stragglers re-evaluate any verdict they touch, and the correction is ledgered (DDIA ch. 11).

**Coverage and connectivity:**
- **Coverage:** `apigen` turns every read operation into an investigation tool. Writes are generated *disabled* until
  a curated action card adds preconditions, an inverse, verification and blast radius.
- **Private connectivity:** PSC/private IP on GCP and PrivateLink/VPC endpoints on AWS; VPC Service Controls
  compatible.

**Relay:**
- A runner plus a NATS leaf node, outbound 443 only.
- A one-time enrolment token, then 24 h certificates from a KMS-backed CA.
- Disconnected sites keep *reading* and buffer their events durably. They hold no write authority; leases lapse and
  their shards show as Unknown.
- **Lease renewal gets its own deadline over the relay.** It has a separate prioritised stream with HTTP/2 PING
  keepalives and an explicit timeout. TCP congestion back-off can silently outlast a lease (Chubby moved
  KeepAlives off TCP for this). It is measured in chaos tests under packet loss.
- Every component degrades rather than crashes when keel or etcd is briefly unavailable: reads continue and writes
  pause.
- Writes and authority are consistent; reads are available and labelled stale (DDIA ch. 9, CAP per operation).

## Console, CLI, API, Terraform

**Resources and layout:**
- **Resources:** `projects/{p}/locations/{site}/fleets/{f}/shards/{s}`, with long-running operations for rehearsals
  and executions.
- **Layout:** project and location selectors, ⌘K, an operations tray. Left nav: Incidents, Fleets, Shards, Tenants,
  Plans, Executions, Approvals, Evidence, Inventory, Changes, Signals, Sites, Connectors, Packs, Policies, IAM,
  Ledger, Trust record.
- **Lists:** dense and virtualised, with an AIP-160 filter bar, a column chooser, saved views, bulk select and an
  Actions menu. Suggestions come from precomputed per-project indexes (Xu ch. 13).

**Incident tabs:**
- **Overview, Scope** (the per-shard position vector, Unknowns), **Proof** (rung badge, raw vs adjusted, knockoff+
  selections with selection frequency, stability with its error bound, assumptions).
- **Plan, Rehearsal** (clone experiments), **Approvals** (the view hash), **Execution** (a live wave grid with an
  always-visible Halt).
- **Verification** (Merkle roots), **Ledger** (inclusion proof, receipt status, head-gossip state), **YAML**.
- Ledger warnings ("head stale beyond MMD") are kept separate from proofs ("two signed heads disagree").

**Labels and behaviour:**
- Visible badges for "mitigation, not repair", "order uncertain" and "not identified".
- Every action shows its effect preview and its CLI, REST and Terraform equivalents.
- Fully keyboard operable ([[console-keyboard-friendly]]).
- Live updates come over SSE with resume offsets.

**CLI and Terraform:**
- **CLI:** `3am shards list --filter=… -o json | 3am probes run --pack accrual --from - | 3am plans draft --from -`;
  `3am plans approve`, `3am executions watch`, `3am halt`.
- **Terraform:** `threeam_site`, `_connector`, `_fleet`, `_policy_rule`, `_approver_group`, `_pack_install`,
  `_autonomy` (pending until quorum).

## Reliability

**HA and DR:**
- Regional GKE/EKS across 3 zones; etcd with 3 voters (5 for banks); NATS R3; Postgres HA.
- keel voters never span regions or clouds.
- Changing from 3 voters to 5 is a scripted, tested procedure: add learners, then a joint-consensus membership
  change. It is never done by hand (Ongaro & Ousterhout 2014).
- Warm DR is a **separate cluster with human-initiated failover**, which halts every plan and reconciles from shard
  markers. Before the standby serves anything:
  - **Its etcd revision is bumped** past the highest fence token held by any shard and past the last revision
    recorded in the ledger (`etcdutl snapshot restore --bump-revision <n> --mark-compacted`). Otherwise every new
    lease's fence would be lower than tokens the shards already hold, and every write would be refused. The bump also
    keeps leader epochs monotonic.
  - **It replays the ledger's tail.** A snapshot can be up to 15 minutes older than the ledger. The standby folds
    every ledgered event after the snapshot into its state and continues the sequence after the ledger's last leaf,
    so it never reuses a sequence number, which would fork the log and be refused by the witness. It refuses to
    serve if the ledger tail can't be read or isn't consistent with the last witness-co-signed head.

**TLA+ (TLC/Apalache in CI), modelling keel *on top of* a linearizable store:**
- PlanLifecycle: TwoPersonRule, drift invalidation.
- WaveExecution: AtMostOnceApply, FenceMonotonic, NoStartAfterHalt, PremiseInTxn, liveness.
- CredentialLifecycle.
- LedgerChain/LogConsistency.
- CdcMerge: the Generalized DBLog contract for the CDC pipeline.
- Each property is classed as safety or liveness. Liveness is **checked**, not only stated, under explicit
  fairness.
- **The invariants are written in TLA+ before code** (M1 step 2). Most errors come from ambiguous English
  (Lamport et al. 2002).
- **The small-instance blind spot.** TLC is feasible only for 2–3 runners and shards. AtMostOnceApply,
  FenceMonotonic and TwoPersonRule therefore get an **inductive invariant** checked by Apalache or TLAPS. TLC random
  simulation covers larger N.
- Third-party parts are modelled at their **documented contract**, as reusable modules: etcd as linearizable CAS
  plus leases and revisions; each engine at its Hermitage-verified isolation; NATS as at-least-once and
  per-partition order. They are never modelled at their internals.
- **Traces flow both ways.** Lab and simulation traces are validated against the specs. TLC counterexamples and
  coverage-directed random traces become scripted madsim schedules.

**Deterministic simulation (madsim)** (Zhou et al. 2021, FoundationDB; Deligiannis et al. 2016, P#):
- 3 keel nodes, N runners, 257 in-memory shards with real transaction, fence and marker semantics.
- madsim can't simulate etcd's internals, so the store is modelled by its contract. The chaos/Jepsen tier covers
  the real store.
- Faults: crashes, partitions, fsync loss, clock skew, slow DBs, **a lost ack after COMMIT**, **a process paused past
  its lease that then tries to commit**, **failover with a lost async tail**, **a late report from a superseded
  epoch**.
- **Buggify points** in keel, runner and `vet` force rare-but-legal behaviour: spurious errors, delays, odd tuning
  values.
- **Swarm testing:** each run randomises voters (3/5), runner and shard counts, fault rates and which buggify points
  are on.
- **Coverage counters** for each named case, with a CI gate requiring each to be reached ≥ N times nightly, so
  "1M seeds clean" can't hide a case never exercised.
- **A PCT-style priority scheduler** runs alongside random seeds. In the P# study, random scheduling alone missed
  4 of 11 bugs.
- **Oracles:**
  - workload invariants: marker count = applied steps; fence monotone; no write without two approvals;
  - **hot/cold liveness monitors**: every halt is reconciled, every Unknown resolves via the marker, every approval
    expires;
  - a **sequential reference model** of "apply plan to shards", compared at each marker commit.
- **Design-fault mutants.** Inject a wrong fence comparison, an off-by-one in scope, or a skipped premise re-check
  into keel and runner, and measure how many `vet` and `auditor` catch. That measures diversity instead of
  assuming it.
- **Halt-and-reconcile is the only error path**, so it is the most-exercised one. Failover code is where the bugs
  live (Chubby).
- Production incidents are reproduced in madsim first, then fixed.
- Millions of seeds nightly.

**Chaos and history checks:**
- Chaos Mesh: kill the keel leader mid-wave, kill a runner mid-commit, SIGSTOP a runner past its lease, partition the
  relay, revoke a credential mid-wave, fail over a shard mid-wave.
- Jepsen-style history checks against markers, fences and checksums.
- **An upgrade drill** next to the failover and restore drills. Chubby's data losses came from software bugs and from
  operator upgrades meant to fix them.

**Adversarial reasoner corpus** (Ruan et al. 2024; Greshake et al. 2023):
- Cases use ToolEmu's schema: underspecification, expected achievement, risky outcomes and risky actions.
- An adversarial evidence emulator plants near-duplicate tenants, shards and payees, similar column names, and
  plausible but wrong probe output.
- **Injection payloads arrive through evidence channels** (DB rows, logs, tickets), not the prompt. They cover
  exfiltration, wrong summaries, multi-stage, encoded and availability attacks.
- Each case is scored on whether verbatim-check, sqlguard, veto or `vet` caught it.
- It runs as a regression suite on every change of model, prompt or pack version.

**Hermitage suite:** isolation anomalies (lost update, write skew, phantoms) are checked per engine and version in
fleetlab, and the result gates the runner.

**Shadow trust record per incident class:**
- Scope precision and recall, DBA-graded cause, plan equivalence (human vs 3AM script, diffed on two clones),
  time-to-scope and time-to-plan, guard catches, the rung reached.
- All reported with CIs.
- Graduation needs, e.g., 30 consecutive runs with 0 unsafe divergences, an upper 95% bound of ≤10% on unsafe rate,
  and ≥95% equivalence. Then lab writes, then production under two-person approval.

## Milestone 1: the 38-of-257 incident

**Lab (`3am-fleetlab`):**
- 257 logical Postgres shard databases across three sites:
  - **GCP:** 2 Cloud SQL instances, ~128 shards;
  - **AWS:** 1 RDS instance, ~64 shards;
  - **"On-prem":** ~65 shards in an isolated VPC with **no inbound rules**.
- Each site has primaries plus a replica, so routing and positions are real.
- A payroll schema (tenants, employees, timecards, pay_runs, pay_lines, accruals, plus **a ledger-style `gl_entries`
  book table**) and a routing directory.
- Go `pay` and `attendance` services on GKE Autopilot with a traffic generator.
- A flag service that buckets tenants by hash.
- Idle instances are stopped.

**Variant catalogue.** Each has its own vaulted ground truth.

| Variant | What it tests |
|---|---|
| `accrual-38` base | Release 2026.10.3 writes bad accrual rows for `accrual_v2` tenants on 38 shards across all three sites |
| `+decoy` | An unrelated fault elsewhere must be excluded |
| `+unreachable` | An on-prem shard is unreachable: Unknown, never clean |
| `+unknown-informative` | Overloaded affected shards time out more often, so missingness correlates with the fault and must be flagged |
| `+co-rolled` | The release and an unrelated flag shipped to exactly the same shards. Statistics must stop at **Supported**, and only the mediator probe or a clone experiment reaches Proven. |
| `+randomised` | The flag rollout was hash-bucketed, so the natural experiment must reach Proven |
| `+simpson` | Raw rates point at the wrong site; stratification must flip the sign and flag it |
| `+bleeding` | The bug is still writing; repair must be blocked until contained |
| `+failover` | A shard fails over mid-wave with a lost async tail; the plan halts and committed shards are re-verified |
| `+celebrity` | One shard carries the largest tenant; it must be ordered last with its own bake |
| `+null` | No fault at all. Over many seeds, false Supported verdicts must stay at or below the FDR level |
| `+no-overlap` | Every shard on release R also has engine v15, and no clean shard does. Must say "no overlap: experiment required", then reach Proven via clone Design A |
| `+threshold` | The flag was rolled out to tenant IDs below a cutoff. Must use regression discontinuity, with a bandwidth sweep, and reach Proven |
| `+noncompliance` | The flag is assigned to 60% of tenants, but only 20% ran a pay cycle. ITT/CACE reported; FAR interval used |
| `+confounded-hidden` | A hidden shared SAN drives both the release wave and the defect. Negative controls or sensitivity (low Γ*) must block Supported; the missing-topology detector proposes the SAN |
| `+post-treatment-trap` | The naive analysis conditions on "shards whose job ran" and flips the sign. `infer` must refuse that adjustment set |
| `+knockoff-calibration` | Strongly correlated null attributes (ρ ≈ 0.3–0.9). Permuted decoys would pass here; model-X knockoff+ must hold FDR ≤ q over many seeds |
| `+repaired-env` | A DBA hotfixed some shards mid-incident. ICP must exclude that environment or use pairwise environments, never name a non-cause |
| `+pg-resurrect` | A Postgres transaction commits in the WAL before it becomes visible. CDC must not resurrect the row it deleted, and verify must not pass on a resurrected row |
| `+sqlserver-skip` | Concurrent updates move rows mid-scan. Chunk reads under locking READ COMMITTED would skip them; SNAPSHOT reads must not |
| `+late-report` | A runner whose lease expired delivers a stale ack after its shard is reassigned. It is quarantined, never merged |
| `+leap-smear` | GCP/AWS smear and on-prem NTP steps across a leap second. Precedence must read "order uncertain", never Proven |
| `+evidence-injection` | Instructions planted in a tenant row, a log line and a ticket. The reasoner's proposal may change, but no gate passes it and nothing leaves the boundary |
| `+relay-loss` | Packet loss on the relay. Lease renewal must keep its deadline or enter jeopardy, and the count audit must find any missing event |
| `+no-mediator` | The faulty release writes no build or version column, as in most real schemas. Rung 4 must come from a mediator source the estate actually has (a tagging hook or server logs joined to CDC transaction IDs), or the verdict must cap at Supported unless rung 5 holds |
| `stuck-flag-38` (operational) | Same scope, but an operational fix exercises the executor end to end |
| `gl-38` (books) | Compensating entries go out as a change package; a human runs it via (a) and (b); 3AM verifies 38/38 |

**Build order (each step gated by its acceptance criteria):**

| # | Step | Acceptance |
|---|---|---|
| 0 | Proto repo, CI, cosign/SBOM, Terraform skeletons for GCP and AWS, ConnectRPC API, `3am version` | CLI ↔ API through Workload Identity / Pod Identity; signed images |
| 1 | Fleetlab + variant catalogue + injector + scorer with vaulted truth | `inject <variant> --seed N` is deterministic; the scorer is the only reader of the truth; idle cost documented |
| 2 | **Spike (1 week): etcd vs FoundationDB for keel.** The tie-breaker is the narrower, better-documented contract, because madsim models the store by contract. Invariants written in TLA+ first. Then keel v0: event-sourced commands with op IDs, leases, epoch, fence = revision, halt, RFC 6962 Merkle ledger with receipts, gossip and witness + legacy verifier, policy golden replay. TLA+ and madsim with buggify, swarm, coverage counters and PCT. Budget: ~2 person-weeks for a focused harness, months for a reusable environment model (P# data). | TLC green; Apalache inductive invariant for the core safety properties; ≥1M seeds clean with every named case's coverage counter reached; liveness monitors cold; 0 divergences replaying `policy.py` (McNemar p = 1); witness refuses an inconsistent head |
| 3 | runner + relay + sqlguard v0 (read-only) on GCP, AWS and on-prem; replica routing with positions; phi-accrual liveness; Hermitage suite for Postgres | 257-shard probe p95 < 15 s and p99 reported; zero inbound on-prem; Unknowns correct; guard corpus of 200+ scripts passes; Hermitage green |
| 4 | Inventory + change streams + rollout system + versioned dimensions + shard/tenant catalogue (Postgres facts and cubes) + CDC under the contract | Release and flag changes visible within 60 s; catalogue lists 257 shards and their tenants; event-time watermarks correct under a simulated partition; `+pg-resurrect` and `+sqlserver-skip` pass; no CDC lock taken; relay count audit clean |
| 5 | Detect + scope + describe + `infer` (rungs 1–3: FRT, negative controls, sensitivity, overlap, IV and RD) + mediator probes (rung 4) | Over all variants × 20 seeds: 38/38 with 0 false positives in < 60 s; rule P = R = 1; knockoffs never above Correlated; `+knockoff-calibration` FDR ≤ q with a CI; `+repaired-env` correct; `+leap-smear` "order uncertain"; stability error bound reported; `+co-rolled` never Proven without rung 4/5; `+simpson` flagged; `+null` false Supported ≤ α with a CI; `+unknown-informative` flagged; `+no-overlap`, `+confounded-hidden` and `+post-treatment-trap` blocked correctly; `+threshold` and `+noncompliance` correct within CI; every verdict's analysis record ledgered before its outcome probes |
| 6 | Draft + sqlguard + rehearsal on clones (Cloud SQL clone, Aurora/RDS snapshot clone, rolled-back on-prem) + clone experiments A/B/C (rung 5) + encrypted before-images + inverse + compensating templates | Counts equal expected 100%; second run 0 rows; inverse restores the checksum; `+randomised` and clone experiments reach Proven with exact p ≤ 0.01; placebo clones never reproduce; designs below the achievable-p bound refused; guard blocks 100% of a mutation suite |
| 7 | Two-person approval (Cedar, WebAuthn; console, CLI, Slack); view hash; operation ID; drift invalidation | Self, author, modified-plan, expired and drifted approvals all refused; approvals and view hashes verify offline from the ledger |
| 8 | Executor: containment gate, interference-group waves, celebrity ordering, fence in DB, premise re-check, marker, `vet` pre-apply check and read-back, jeopardy state, JIT credentials (GCP conditional IAM, AWS session policy, OpenBao), halt, verify with Merkle roots | **38/38 verified, 219 untouched**. Chaos runs (leader kill, runner kill, SIGSTOP past lease, credential revoke, `+failover`, `+late-report`, `+relay-loss`) each resume or halt with **0 double-applies and 0 stale-fence commits**. `+bleeding` blocked. No credentials left behind. `vet` catch rate on design-fault mutants reported. |
| 9 | Books path: change package (a) and `3am-run` (b) on `gl-38` | The human-executed run is verified 38/38 by 3AM; the ledger shows the executor and key; 3AM never held a book credential |
| 10 | Auditor: scheduled re-verification, ledger ↔ marker ↔ keel cross-check from raw sources, receipt inclusion checks, projection rebuild diff, restore and upgrade drills | A regression injected after a verified fix opens an incident; a withheld receipt and a split-view head are each detected; drills restored and ledgered |
| 11 | Follow-through: Jira defect + PR (fix + regression test) on the lab `pay` repo | PR CI green, linked from the incident through the operation ID |
| 12 | Console + CLI for the whole flow | Full demo from the console alone and from the CLI alone, keyboard only |
| 13 | Shadow harness + trust report + adversarial reasoner corpus | Plan equivalence by clone diff; every rate with a CI; IPW estimate for one proposed autonomy rule; `+evidence-injection` passes; 0 corpus cases reach a credential |

**End-to-end target:** injection → verified in under 15 min, excluding the time humans take to approve.

**Budget, schedule and staffing:** `docs/superpowers/plans/M1-BUDGET-AND-STAFFING.md` adds up every step's cost,
lab time and people. No step applies Terraform before the founder sets the monthly cap (open question 12).

**Deferred to M2+:**
- SQL Server and Oracle (JDBC sidecar), unless the design partner is on SQL Server; then they move into M1.
- Spanner and AlloyDB actions; DDL plans.
- Marketplace listings; full generated API coverage.
- The Terraform provider (resource model frozen in M1).
- The ClickHouse large-estate tier.
- TeamCity; cross-region DR drills; FIPS and air-gapped packaging.
- L2 autonomy on fleets.

## Capacity (large customer: ~10k hosts, 100k resources, 257 DB servers; peak = 2× average, Xu ch. 2)

| Item | Estimate (assumption) |
|---|---|
| Inventory | 500 MB (Postgres) |
| Change events | ~2M/day avg, 4M peak → ~5 GB per 90 days (Postgres partitions) |
| Derived signals | ~150k series at 1 min → 2.5k pts/s avg, 5k peak ≈ 0.5–1 GB/day → ~300 GB per 13 months. Postgres partitioned; ClickHouse only beyond ~1M series or ~50k pts/s |
| Steady probes | ~17 queries/s fleet-wide, ≤k per server |
| Incident fan-out | 2.6 MB per incident; p50 ~10 s, p99 bounded by per-shard deadline (Unknown beyond it) |
| Clone experiments | 1 clone per affected DB server per rung-5 test; minutes, billed to the customer's project |
| Bus | < 10 MB/s peak on NATS |
| CDC slots | One logical slot per shard database (257 in the lab, 64 per server); an aggregate WAL budget per server (default 20 GB); decoding overhead measured per server |
| keel | ~100 transitions/s peak; etcd comfortably below its write ceiling |
| Ledger | 500 MB/day, ~1.3 TB per 7 years, cold and locked; Merkle proofs O(log n) |
| Before-images | MB to hundreds of MB per incident, crypto-shredded at 90 days by default |
| 3AM's footprint | ~1e-5 of customer data; nothing raw moves |

The binding constraint is cloud monitoring and logging read quotas, handled by pre-aggregation and per-project token
buckets.

## Critical files to carry over (specification sources)

- `~/super-repo/3am/core/threeam/policy.py`: rule semantics for keel, with golden replay.
- `~/super-repo/3am/core/threeam/connectors/sql.py`: row caps, rolled-back dry run and no DDL rehearsal become runner
  invariants.
- `~/super-repo/3am/core/threeam/ledger.py`: the chain and seal format the Merkle log must keep verifying.
- `~/super-repo/3am/core/threeam/decision/loop.py`: the resolution state machine for the PlanLifecycle/WaveExecution specs.
- `~/super-repo/3am/core/threeam/investigate_llm.py`, `systemone.py`: reasoner rules (reads only, verbatim values, veto-only).
  `systemone.py`'s veto becomes a loss-matrix Bayes rule with a reject option.
- `~/super-repo/3am/packs/*/pack.json`: input to the `pack.v2` converter.
- `~/super-repo/3am/console/src/keys.tsx`, `query.ts`: the keyboard map and query language to keep.
- `~/super-repo/3am/build/eval_loop.sh`: regression harness, run against classic and the new engine during the strangler.

## Verification

- **Each M1 step:** the acceptance criteria above, run in `3am-fleetlab` across GCP, AWS and on-prem.
- **Safety:**
  - TLC/Apalache green on every spec;
  - nightly madsim ≥1M seeds with no invariant violation;
  - Chaos Mesh runs with 0 double-applies and 0 stale-fence commits (history check against markers, fences and
    checksums);
  - Hermitage green per engine;
  - 100% sqlguard mutation-suite block rate;
  - 0 divergences on golden replay of classic ledgers;
  - the auditor's cross-check clean.
- **Statistics:** the `+null` false-Supported rate ≤ α (with a CI); knockoffs never promoted and
  `+knockoff-calibration` FDR ≤ q; stability error bound reported; grouped CV by scenario family; vaulted truth read
  only by the scorer.
- **Regression:** Pinata's `build/eval_loop.sh` on classic and new engines; the demo set (6, 11, 14, 97) passes
  every release ([[demo-set]]).
- **The milestone demo:**
  - `fleetlab inject accrual-38+decoy+unreachable` → scope 38/38 in < 60 s → rule and proof (rung 4, then rung 5 on
    clones) → containment → rehearsal → two-person approval → waves → "38/38 verified, 218 untouched, 1 unknown" →
    Jira + green PR;
  - `gl-38` via `3am-run` under a human key;
  - repeated from the console alone and from the CLI alone.

## Failure modes and how each is closed

| Failure | Closed by | Proven by |
|---|---|---|
| Fix applied twice (retry, lost ack, failover replay) | Marker in the same transaction; timeout = unknown | TLA+ AtMostOnceApply; madsim lost-ack; chaos history check |
| Stale runner writes after losing its lease (GC/VM pause) | Fence checked by the customer DB itself | madsim and chaos SIGSTOP-past-lease: 0 stale commits |
| Data changed between approval and execution | Premise re-check `FOR UPDATE` in the transaction; drift invalidates approval | Mutation tests; lab drift injection |
| Bug still writing while we repair | Contain-before-repair gate | `+bleeding` variant |
| Failover silently drops a committed fix | Halt on timeline change; re-verify by marker | `+failover` variant; madsim async-tail loss |
| Replica lag makes a broken shard look clean | Position floor; verify at ≥ commit position or on primary | Lab lag injection |
| Unreachable shard assumed clean | Unknown never imputed; bounds; informative-missingness test | `+unreachable`, `+unknown-informative` |
| Wrong tenant/shard routing | Routing-directory version pinned per plan; move = drift | Lab tenant move mid-plan |
| Script widens beyond scope (OR, missing tenant key, nondeterminism, triggers) | sqlguard AST rules, EXPLAIN, rowcount cap, declared side effects | 200+ script corpus; 100% mutation-suite block |
| Isolation anomaly under live traffic (lost update, write skew) | Atomic set-based UPDATE; per-engine isolation; Hermitage gate | Hermitage suite per engine/version |
| Wrong cause: confounding, Simpson, co-rollout, hidden common cause | Proof ladder; overlap gate; negative controls; sensitivity Γ*; only Proven executes | `+simpson`, `+co-rolled`, `+confounded-hidden`, `+no-overlap` |
| False discovery from searching hundreds of attributes | Model-X knockoff+ selection; permutation null; sample splitting; stability selection with stated bound; pre-registered analysis record | `+null` and `+knockoff-calibration` over many seeds; knockoff promotion = 0 |
| Decoy check passes while correlated nulls slip through | Knockoffs preserve cross-correlation; never permuted copies | `+knockoff-calibration` |
| ICP names a non-cause (selection, repaired shards, outcome-defined environments) | Exogenous environments only; informative Unknowns cap rung 3; pairwise environments | `+unknown-informative`, `+repaired-env` |
| CDC silently misses or resurrects a row | CDC contract per engine; snapshot comparison on Postgres; SNAPSHOT on SQL Server; fail closed on discontinuous history | `+pg-resurrect`, `+sqlserver-skip`; CdcMerge spec |
| Stale report from a superseded epoch flips state | Epoch on every report; quarantine (invariant 15) | `+late-report`; madsim superseded-epoch case |
| keel command applied twice by an etcd client retry | Op ID inside the same etcd transaction | madsim lost-ack on keel commands |
| Lease renewal starved by TCP back-off on the relay | Separate prioritised keepalive with its own deadline; jeopardy | `+relay-loss`; chaos under packet loss |
| Correlated design fault in keel and runner | `vet` diverse pre-apply check with a separate owner and raw inputs (invariant 17) | Design-fault mutant catch rate |
| Conditioning on post-change variables | Covariate rules refuse post-treatment adjustment sets | `+post-treatment-trap` |
| LLM hallucinated value or plan | Reasoner can't grant rungs; values verbatim from evidence, checked mechanically; ambiguity fails; System 1 veto; sqlguard | Golden replay; adversarial reasoner corpus |
| Evidence contains instructions to the reasoner (indirect prompt injection) | Invariant 16; no egress; no memory; templated tools; hidden-text stripping; deterministic evidence first in the approval view | `+evidence-injection`; corpus with injections through evidence channels |
| Fix treats symptom, not cause | "Mitigation, not repair" label; verify cause probe | Lab symptom-only plans can't close |
| Canary "looked fine" by chance | Randomised canary + FRT health gate with known false-pass rate | Lab regression injected into a later wave |
| Rubber-stamp approval | Two distinct humans, WebAuthn, view-hash, expiry, Cedar | Refusal tests; ledger audit |
| Insider rewrites history | Transparency log with customer witness co-sign; inclusion/consistency proofs | Auditor cross-check; external verification tool |
| 3AM shows different histories to different parties (split view) | Head gossip with consistency proofs; witness stores heads and refuses inconsistent ones | Lab split-view injection |
| keel accepts a record but never ledgers it | Receipts with an MMD; inclusion proof demanded per receipt; a fresh head every MMD | Lab withheld-receipt injection |
| "Deleted" data survives on SSD, snapshot or swap | Crypto-shredding everywhere plaintext can land; ledgered key destruction | Retention drill; runner disk and swap scan |
| Compromised site forges evidence | Per-site signing keys; range-checked summaries; no BFT claim (books stay human) | Pen test |
| AI writes to books | No book credential exists; sqlguard refusal; `3am-run` runs under human key only | `gl-38`; credential inventory audit |
| PII leak via evidence or prompts | Summaries only; per-incident keys; crypto-shredding; no PII to external models | DLP scan in CI; retention drill |
| Clock skew fakes "change preceded onset" | Clock-uncertainty intervals with measured ε; `C.latest < O.earliest`; NTP gate ±250 ms; three timestamps; LSN order within a shard | Skew injection in madsim; `+leap-smear` |
| Schema/version mismatch during upgrade | `critical` fields rejected by old readers; servers first; buf breaking | Mixed-version lab runs |
| Coordinator down mid-wave | etcd quorum; halt-safe; leases lapse; reconcile from markers | Chaos leader kill |
| Fix later undone by a new release | Auditor scheduled re-verification opens incident | Post-fix regression injection |
| Cloud quota exhaustion blinds detection | Token buckets per project; pre-aggregation; degraded mode labelled | Quota-limit lab test |
| Payroll window contention | Declared windows; PK batches < 1 s; lock_timeout | Load test at peak replay |

## Residual risks (what this plan cannot prove away)

- **Identification can fail silently.** Observational verdicts rest on untestable assumptions. That is why only
  rungs 4–5 execute, and why clone permission (open question 5) matters so much.
- **The SUTVA assumption inside a clone** (Design A) is assumed and checked indirectly, not proven.
- **Provability at a real customer.** Rung 4 needs a row-level mediator, and rung 5 needs either a randomised
  rollout or a replay harness that runs the release against a clone. The lab has both by construction (a
  `writer_build` column and fleetlab's own `pay` worker). Many real estates have neither, and there most verdicts
  stop at Supported, so nothing executes. Open questions 13 and 14 decide what each estate can prove; `+no-mediator`
  tests the fallback.
- **Lab validity is internal only.** Every customer estate needs its own shadow trust record before any write.
- **Correlated software faults.** The same code runs in every site, so a keel/runner bug could be fleet-wide.
  - **Tolerated at execution time** by `vet`, the diverse pre-apply checker.
  - **Detected after the fact** by `auditor`.
  - Reduced by deterministic simulation, staged rollout of 3AM itself, and the books boundary.
  - Design diversity is not independence. Shared specs, shared people and shared libraries cause related faults,
    and independently written versions have been measured failing together far more often than independence
    predicts (Avizienis 1985; Knight & Leveson 1986). So `vet`'s protection is measured with design-fault mutants,
    never computed by multiplying failure probabilities.
  - `vet` and keel must run the same spec version, and a deploy with mismatched versions is refused.
- **Exchangeability is assumed** by conformal calibration and stability selection. Payroll seasonality and deploys
  break it. Mondrian splits and recalibration reduce the damage but don't remove it.
- **Prompt injection has no complete defence.** The plan's answer is structural: the reasoner holds no authority and
  no egress. A manipulated proposal can still waste human time.
- **Customer-side changes we can't see** (manual DBA hotfixes outside 3AM) show as drift or unknown, never as
  clean; they can still delay incidents.
- **Human approvers can still approve a wrong-but-proven plan** if the proof's estimand doesn't match the
  business question; the analysis record and view-hash make this auditable, not impossible.

## Production gates (no customer write until every one is green)

1. TLA+ specs green; ≥1M madsim seeds nightly clean for 30 consecutive nights.
2. Chaos suite: 0 double-applies, 0 stale-fence commits across all scenarios.
3. Hermitage green for the customer's exact engine/version.
4. sqlguard 100% mutation block; external review of the rule set.
5. Fleetlab: all variants pass with stated CIs; `+null` false-Supported ≤ α.
6. Independent security review + pen test of keel, runner, `vet`, relay, broker; ledger verifiable by a customer-run
   tool (RFC 6962-compatible); witness refusal and head gossip demonstrated.
7. Customer estate shadow record meets graduation (30 consecutive, 0 unsafe, upper 95% bound ≤10%, ≥95%
   equivalence).
8. Customer DBA sign-off on marker/fence tables, isolation settings, payroll windows; CAB integration if required.
9. Restore and upgrade drills of ledger and evidence store passed and ledgered.
10. CDC contract tests green for the customer's exact engine/version (`+pg-resurrect`, `+sqlserver-skip`).
11. `vet` design-fault mutant catch rate reported and accepted; adversarial reasoner corpus with 0 cases reaching a
    credential.

## Open questions (need the founder before step 3)

1. The design partner's engine(s) and versions, and the tenancy model (database, schema or `tenant_id` per tenant).
2. The source of truth for tenant → shard routing, and whether it is versioned.
3. Will DBAs accept `_threeam.applied` and `_threeam.fence` (with its `ops` and `books` lanes) in each shard, plus
   one logical replication slot per shard database? They are the exactly-once and fencing anchors, and the CDC
   source. Postgres CDC needs no 3AM-owned watermark table (it uses read-only LSN brackets); another engine may.
4. Where may encrypted before-images (payroll PII) live, and for how long? The proposed answer: encrypted at source
   and deleted by ledgered key destruction, so location matters less.
5. May rehearsal and **clone experiments** use production clones, or only replicas/staging? Rung 5 depends on this.
6. Are the design partner's flag and release rollouts **randomised** (hash-bucketed)? If so, natural experiments come for free.
7. Are their book tables ledger-style (compensating entries) or mutable balances?
8. Approver identity: Okta or Entra? Hardware keys? Is a ServiceNow CAB step required?
9. On-prem secrets: CyberArk, Vault, or nothing?
10. Must on-prem sites keep reading if the control plane is down? (Planned: yes, read-only.)
11. Which LLMs are allowed inside the boundary: self-hosted, Vertex or Bedrock? Decide this on PII and residency
    grounds. Injection risk is the same for all of them, and the structural controls handle it. Until it is
    decided, the lab calls models through a regional or multi-region endpoint (`us`, or `eu`), never `global`,
    which may process anywhere.
12. The monthly lab budget (GCP + AWS), including clone-experiment cost. It gates every Terraform apply; the
    roll-up is in `docs/superpowers/plans/M1-BUDGET-AND-STAFFING.md`.
13. **Which row-level mediators does the estate have?** A build or version column written by the application; a
    tagging hook the customer would install (the application sets a session tag and a 3AM-owned trigger records it
    per row); database server logs (pgaudit or `log_line_prefix` with `application_name` and the transaction ID)
    that can be joined to CDC transaction IDs; or none. With none, rung 4 is unavailable.
14. **Can the customer provide a replay harness?** That is a signed image of the faulty and prior releases that runs
    one job or cycle against a clone, with a configuration override for the intervention (the replay contract in
    `3am-packs`). Without it, rung 5 by clone experiment is unavailable, and only natural experiments (randomised or
    threshold rollouts) remain.

## Housekeeping after approval

- This plan is published as `PLAN.md` in https://github.com/3am-si/cortex. Keep it in sync on each revision.

- Add `platform-revamp.md` to `MEMORY.md`.
- Update `3am-web-design-system.md`: the console now follows the AWS/GCP console pattern (functionality over design),
  superseding the 2026-10-06 "light enterprise" note.

## Book index: principle → where it lives

### System Design Interview Vol. 1 (Xu)

| Ch. | Principle | Lives in |
|---|---|---|
| 1 | Stateless tiers; sharding hot spots ("celebrity"); no cross-shard joins | Architecture (stateless `control`); Execute (celebrity last); Scope (summaries merged centrally) |
| 2 | Back-of-envelope, peak = 2× avg, state assumptions | Capacity |
| 4 | Token buckets, typed 429 | Cloud (read-quota budgets); keel write budgets; `RESOURCE_EXHAUSTED` with retry-after |
| 5 | Consistent hashing | Superseded by DDIA ch. 6 (fixed assignment table in keel) |
| 6 | Quorums, Merkle anti-entropy | keel stays CP; Verify (Merkle roots prove "219 untouched") |
| 7 | Time-sortable IDs | UUIDv7; plan order by etcd revision |
| 9 | Crawler politeness, priority queues, blackout windows | Scope (k per server, priority queue); Execute (payroll windows) |
| 10 | At-least-once notification, dedupe key, per-channel queues | Follow-through (after-commit, keyed by outcome ID) |
| 11–13 | Fan-out hybrid; heartbeat grace; precomputed top-k | SSE watch with resume offsets; liveness (now phi-accrual); console suggestions |
| 14 | DAG scheduler; pre-signed URLs; retryable vs terminal errors | Pipeline intro; runner uploads |
| 15 | Content-addressed blobs; first-writer-wins metadata; offline queue; cold tiering | Evidence store; `plan_hash` CAS; relay buffering; ledger cold storage |

### Designing Data-Intensive Applications (Kleppmann)

| Ch. | Principle | Lives in |
|---|---|---|
| 1 | Faults vs failures; self-checking; human error; percentiles and tail amplification; operability | Audit (self-audits); per-shard deadlines and HDR histograms; console effect previews |
| 2 | Declarative languages; property graph; Datalog | `pack.v2`; topology graph; Datalog scope predicates |
| 3 | OLTP vs OLAP; column stores, cubes; append-only logs, torn tails | Scope (index-proven probes on replicas); Postgres cubes (ClickHouse optional); ledger segments |
| 4 | Schema evolution; RPC ≠ local call; brokers | Protocols (`critical` extension, descriptor hash); invariant 9 |
| 5 | Replication lag; failover loses writes; statement nondeterminism; logical CDC; one leader | Scope (position vector); invariants 11, 13; CDC; keel single leader |
| 6 | Partition map as versioned input; local vs global index; human-in-loop rebalancing | Approve (routing version); global index used only to narrow probes; fixed assignment table |
| 7 | Retries and lost acks; weak isolation; write skew; phantoms; short transactions | Invariants 5, 7, 4; sqlguard per-engine isolation and Hermitage; PK batches < 1 s |
| 8 | Partial failure; phi accrual; fencing in the resource; clocks as intervals; system model | Invariant 6; Degraded runners; Prove rung 4 clock intervals; safety vs liveness |
| 9 | Linearizability; CAP per operation; don't build consensus; no 2PC; static membership | etcd keel; invariant 8; DR as a separate cluster; relay semantics |
| 10 | Move compute to data; build-then-swap; immutable inputs; determinism; Unix composition | Scope (map/reduce); Draft (after-images); derived state as versioned builds; replayable `infer`; CLI pipes |
| 11 | Commands vs events; no dual writes; compensating entries; streams vs queues; event time; time-dependent joins; effectively-once | keel event sourcing; invariant 12; books templates; bus; watermarks; versioned inputs; invariant 7 |
| 12 | Few moving parts; coordinate only the irreversible; end-to-end op ID; reads as events; trust but verify; transparency logs; data as hazmat; accountability | Required core; Architecture intro; operation ID; view hash; `auditor`; Merkle ledger with witness; crypto-shredding; invariant 14 |

### Elements of Causal Inference (Peters, Janzing, Schölkopf)

| Ch. | Principle | Lives in |
|---|---|---|
| 1, 3, 6 | Observation can't identify causes; P(Y \| X) ≠ P(Y \| do X) | Proof ladder (only rungs 4–5 are Proven) |
| 2 | Independent mechanisms; sparse mechanism shift | RCA as mechanism-shift localisation; bet on sparsity |
| 3, 6 | SCMs as programs; deploys and flags as interventions | Topology mechanism nodes; change streams |
| 6 | Backdoor adjustment; Simpson; Berkson; front door; IV and randomisation; SUTVA | Rung 2; Scope (whole population); rung 4; rung 5 natural experiments; interference groups |
| 6.8 | Interventions on clones are falsification tests | Rehearse (clone experiments, prediction falsification) |
| 7, 8 | ICP across environments; half-sibling regression; IPW off-policy evaluation; label shift | Rung 3; Detect; Learning (autonomy, priors) |
| 9 | Hidden confounders; PAG wording; low-rank fingerprints | Assumption statements; missing-topology detector |
| 10 | Granger needs sufficiency and fine sampling | Rung 1 only |

### Elements of Statistical Learning (Hastie, Tibshirani, Friedman)

| Ch. | Principle | Lives in |
|---|---|---|
| 2, 4 | Bayes rule with a loss matrix; reject option; curse of dimensionality | Detect thresholds; abstain; System 1 veto; no kNN on raw metrics |
| 3 | Correlated predictors are unstable; lasso/elastic net/group lasso; irrepresentable condition; inference after selection is invalid | Honesty rules (co-rolled suspects; sample splitting) |
| 5, 6 | Natural splines and local-linear fits fix boundary bias at "now"; additive seasonal baselines; mixtures | Detect; pack signatures as mixtures |
| 7 | Optimism and effective df of search; CV estimates expected error; the wrong way to CV; one-SE rule; report SEs | Permutation null; grouped CV inside folds; vaulted truth; CIs everywhere |
| 8 | Bootstrap includes adaptive steps; bootstrap ≈ posterior; bagging; stacking; BIC posteriors; EM | Stability bootstrap; detector stacking; hypothesis posteriors |
| 9 | GAMs; trees and PRIM as interpretable rules; high-cardinality overfit; loss-weighted trees; ROC; MAR/MCAR | Describe (rule); invariant 10 and `+unknown-informative` |
| 10, 15 | Robust (Huber) loss; hurdle models for counts; importance and partial dependence are not causal; OOB ≈ CV | Detect baselines; bad-row counts; honesty rule on interpretation |
| 11 | Artificial probe features expose spurious selection | Knockoffs in every differential (built as model-X knockoffs, not permutations; see the paper index) |
| 12, 13 | Hinge loss doesn't give probabilities; several prototypes per class; invariant metrics | Calibrated probabilities only; mixture signatures; incident similarity |
| 14 | Unsupervised has no ground truth; gap statistic / permuted null; unsupervised-as-supervised drift; product-of-marginals reference; ICA | Clustering as navigation; missing-topology null; drift classifier; rare associations |
| 16 | Bet on sparsity; rule ensembles | Estimator choice; Describe |
| 17 | Graphical lasso: conditional-independence graphs | Candidate topology edges (undirected) |
| 18 | p ≫ N needs heavy regularisation; nearest shrunken centroids; fused lasso for change points; permutation FDR, BH, q-values, local fdr | `infer` estimators; onset and PK-run boundaries; differential significance |

### A First Course in Causal Inference (Ding)

| Ch. | Principle | Lives in |
|---|---|---|
| 1–2 | Yule–Simpson; specification search; potential outcomes; no causation without manipulation; SUTVA; before/after ≠ two units | Analysis record (well-defined intervention, unit); pre-registered specs; DiD verification |
| 3–4, 8 | Fisher randomization test exact for any statistic; Neyman conservative variance; studentized FRT valid under both nulls; Phipson–Smyth p-values; show randomization distribution | `infer` design-based rules; console proof tab |
| 5–7 | Block what you can, randomize what you cannot; rerandomization; Lin adjustment; matched pairs; minimum units for a reachable p | Clone Designs A/B; randomised canaries; achievable-p check |
| 9 | Finite vs super population; internal vs external validity | Fleet = finite population; per-estate shadow record |
| 10–15 | Identification; ignorability; propensity score, balance checks before outcomes; IPW/Hajek; doubly robust; ATT; matching | Estimand = ATT; outcome-blind design; AIPW/matching estimators |
| 16 | Negative outcomes/exposures; M-bias and Z-bias from over-adjustment | Mandatory negative controls; covariate rules |
| 17–19 | E-value; sensitivity parameters; Manski bounds; Rosenbaum Γ | Sensitivity headline; bounds for Unknowns; Γ* ≥ 2 for Supported |
| 20 | Overlap tension; regression discontinuity, bandwidth sensitivity, McCrary | Overlap gate; RD for threshold rollouts and deploy instants |
| 21–25 | Encouragement/noncompliance, CACE, weak IV, FAR intervals, IV inequalities, fuzzy RD, Egger regression | Rollout analysis (ITT/CACE); falsification; multiple instruments |
| 26–28 | Principal stratification; never condition on post-treatment variables; natural effects are cross-world; controlled direct effect | Covariate rules; mechanism via cde in clones |
| 29 | Time-varying confounding; g-formula, MSM, sequential IPW; g-null paradox | Autonomy/remediation policy evaluation |

## Paper index: paper → what it changed

Full reading notes are in `papers/NOTES.md`. The folder is gitignored because the PDFs are copyrighted.

| Paper | Changed | Lives in |
|---|---|---|
| Crosby & Wallach 2009, tamper-evident logging | Tamper evidence needs audits and gossip; Merkle aggregation for provably complete queries; batched signing | Ledger |
| Gutmann 1996, secure deletion | Overwrite isn't deletion on SSDs or clouds; crypto-shred everywhere plaintext lands; ledger key destruction | Data is hazardous; open question 4 |
| Zhou et al. 2021, FoundationDB | Buggify, swarm testing, coverage counters; store modelled by contract; one tested error path; short chunked transactions | Reliability; sqlguard; step 2 |
| Ongaro & Ousterhout 2014, Raft | Idempotent keel commands by op ID; linearizable authority reads; CAS on the leader revision; learners and joint consensus | Invariant 12; keel; HA |
| Helland 2015, immutability | Compensating entries; version IDs never mutable names; clones as immutable DataSets | Books; Approve; Rehearse |
| Barber & Candès 2015, knockoffs | **Permuted decoys fail FDR (45.6% at a nominal 20%)**; knockoff+; BH not guaranteed under correlation | Honesty rules; `+knockoff-calibration` |
| Shafer & Vovk 2008, conformal prediction | Distribution-free alert rates; Mondrian calibration; credibility as the reject option; exchangeability caveat | Detect; residual risks |
| Corbett et al. 2012, Spanner | Precedence = `C.latest < O.earliest`; measured ε; leap-smear mismatch; LSN order; lease length from halt latency | IDs and time; `+leap-smear` |
| Burrows 2006, Chubby | Fence = sequencer; lock-delay for unfenceable targets; jeopardy; epochs; suffix-only loss; keepalives vs TCP back-off | Invariant 6; runner; relay; Execute |
| Fischer, Lynch & Paterson 1985, FLP | Basis of TimeoutIsUnknown; no safety property on timeouts; liveness under partial synchrony | Invariant 9; liveness |
| Lamport et al. 2002, TLA+ | Invariants in TLA+ first; inductive invariant for large-instance bugs; traces both ways; contract modules | TLA+ |
| Deligiannis et al. 2016, P# | Hot/cold liveness monitors; PCT scheduler; reference-model oracle; stale reports from dead nodes | Invariant 15; madsim; `+late-report` |
| RFC 6962, Certificate Transparency | Exact tree construction; receipts with an MMD; a fresh head every MMD; key-version history | Ledger; gate 6 |
| Chuat et al. 2015, CT gossip | Head + consistency-proof gossip in existing RPCs; witness refuses inconsistent heads; warning vs proof | Ledger; console |
| Candès et al. 2018, model-X knockoffs | Knockoffs from the attribute distribution; CRT = FRT for randomised attributes; no logistic p-values at p/n ≳ 0.1; group knockoffs; i.i.d. caveat | Honesty rules |
| Peters, Bühlmann & Meinshausen 2016, ICP | **Rung 3 = "contained in the parents"**; exogenous environments; informative Unknowns cap; pairwise environments | Rung 3; ICP rules; `+repaired-env` |
| Meinshausen & Bühlmann 2010, stability selection | Error bound; complementary pairs; group and block resampling | Honesty rules |
| Andreakis & Papapanagiotou 2020, DBLog | No locking snapshots; non-stale reads vs replicas; watermark table is a write (Postgres uses read-only LSN brackets instead) | CDC contract; open question 3 |
| Andreakis 2026, Generalized DBLog (preprint, machine-checked) | **Postgres resurrected-delete fix**; SQL Server snapshot reads; `updated_at` is outside the contract; coordinate-bound probes; fail closed | CDC contract; Scope; rung 4; `+pg-resurrect` |
| Kreps et al. 2011, Kafka | At-least-once, so idempotent consumers; per-partition order; retention alarms; count audits; schema IDs | Bus contract |
| Saltzer & Schroeder 1975, protection | Revocation latency ≥ TTL, per platform; prescripts; complete mediation; confinement of the reasoner; no shared approver accounts | JIT; Approve; invariants 14, 16 |
| Ruan et al. 2024, ToolEmu | Prompting isn't a control (23.9% failures); mechanical verbatim and ambiguity checks; LLM judges advisory; adversarial corpus | LLM boundary; reasoner corpus |
| Greshake et al. 2023, indirect prompt injection | Evidence is attacker-controlled; no egress; no memory; hidden-text stripping; evidence above prose | LLM boundary; invariant 16; `+evidence-injection` |
| Avizienis 1985, N-version software | **The auditor detects, it doesn't tolerate**; `vet` in the commit path; spec as the common cause; measure diversity | Invariant 17; `vet`; residual risks |

Outside the reviewed set, cited once: Knight & Leveson 1986 (an experiment showing correlated failures across
independently written versions), Shah & Samworth 2013 (complementary-pairs stability selection).
