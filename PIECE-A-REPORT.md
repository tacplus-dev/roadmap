# Accounting durable sink — Piece A report (AccountingRecord proto + regen)

Date: 2026-08-25. Scope: **Step 1 of the 5-step plan only** (design doc §3 step 1).
Nothing from Piece B (emitter wiring) or Piece C (Rust identity fix, `request_id` on
`TaskActivity`, e2e rewrite) is implemented. `testapp/e2e/run.sh` was not touched.

## What changed (uncommitted, as instructed)

Two files, both in `platform/proto` (the only git repo touched; `tacplus` and `testapp`
are `git status` clean):

```
 M registry.yaml                        (1 entry added by the tacplus tool, not by hand)
?? vservices/accounting-record.proto    (new)
```

Everything else that moved is generated output under ignored/untracked `target/` and
`model/` trees, produced by `testapp/build.sh`.

## Q6 — RESOLVED: there IS an allocation tool. No manual edit was made.

The design doc's Q6 assumed a tool existed but couldn't find it. It exists:

- `tacplus registry --proto-dir <dir>` — `tacplus/cmd/registry/cmd.go`. It scans
  `services/` for `(spec.service)` and `vservices/` for `(spec.vservice)` (via
  `registry.ScanVServiceNames`, `tacplus/pkg/registry/registry.go:330`) and assigns any
  unregistered name the next sequential ID (`Registry.VServiceFieldID`, `registry.go:138`;
  `nextID()` at `:242` takes `max(all service ids, all vservice field_ids) + 1`, ceiling 99).
- Its `Short` string is literally "Update the service/vservice registry from proto
  definitions", and its `Long` says "Run this before tagging the proto repo to ensure all
  services and vservices have stable, non-overlapping field IDs."

Run, and its actual output:

```
$ ./testapp/.tacplus-bin/tacplus registry --proto-dir platform/proto
INFO  loaded registry  {"path": "platform/proto/registry.yaml", "services": 17, "vservices": 2}
DEBUG found vservice declaration  {"name": "accountingrecord"}
DEBUG found vservice declaration  {"name": "agentsensor"}
DEBUG found vservice declaration  {"name": "proxypolicy"}
...
VServices (3):
  agentsensor          = 42
  proxypolicy          = 45
  accountingrecord     = 52
```

Resulting diff — the whole of the registry change, written by the tool:

```diff
 vservices:
     - name: agentsensor
       field_id: 42
     - name: proxypolicy
       field_id: 45
+    - name: accountingrecord
+      field_id: 52
```

52 is `max(51 = samlsvc) + 1`, exactly what `nextID()` specifies. **The "Managed by
tacplus — do not edit IDs manually" header is accurate, not aspirational.** Note the tool
is *not* wired into `testapp/build.sh`; it is a separate step you run when adding a
vservice. (`tacplus build` will also auto-assign via `configholder.go:755`, but against the
temp/remote proto checkout, so the ID would not persist — running `tacplus registry`
against the real `platform/proto` is what makes it stable.)

## Two design-doc corrections, both found by the build failing

### 1. Q1 is wrong: stratum DOES have a retention knob, and it is MANDATORY.

The design doc says "There is no retention knob on a stratum record … stratum's record
store cannot express [retention] today." That is false, and the first stratum build failed
on it:

```
[ERROR] Failed to execute goal ... stratum-codegen:generate-vservice-wiring ...:
  accountingrecord.AccountingRecord is stored by stratum but declares no
  (spec.timeseries_projection).retention.timestamp_field, so the retention sweep has no
  field to measure event age from. Declare a retention policy on the record in
  vservices/accounting-record.proto.
```

The source of truth is `platform/services/stratum/codegen/.../EventStoreGenerator.java:195`
(`retentionAttribute`): stratum's scheduled Hazelcast sweep reads
`(spec.timeseries_projection).retention.timestamp_field` per record to know which field
carries age, and **rejects at build time any stored record that declares none** ("could
never be swept, so it is rejected here rather than generated into a sweep that cannot
work"). So a record's retention policy lives on its timeseries projection, and every
stratum-stored record must have one.

This also settles the design doc's Q3 by construction: `AccountingRecord` carries a
`timeseries_projection` (`AccountingRow`, 90-day retention — the same 7776000s window
`AgentEvent` and `TaskActivity` use) because it cannot be stored without one. Whether to
*also* thin the full-record set to ERROR/WARN only remains a genuine open question.

### 2. New tacplus defect found: Rust projection generator drops cross-package enum imports.

Projecting the `level` field (type `accounting.EventLevel`, a **cross-package** enum) into
the timeseries projection generates Rust that does not compile:

```
error[E0433]: cannot find type `EventLevel` in this scope
  --> testapp/services/tasks/target/generated-sources/model/types/vservices/accountingrecord/timeseries_accounting_row.rs:55:20
55 |    level: EventLevel::try_from(msg.level).map(|e| e.as_str_name().to_string())...
   |           ^^^^^^^^^^ use of undeclared type `EventLevel`
help: consider importing this enum through its public re-export
   |
22 + use crate::sys::accounting::EventLevel;
```

The mapper emits `EventLevel::try_from(...)` but never emits the `use`. A same-package enum
(`TaskActivity.status` → `TaskStatus`) is unaffected, which is why nothing had hit this.
This surfaced as `build.testapp FAIL(regression)` in the harness — a real regression I
introduced, not a flake.

**Workaround taken (stated plainly, since it is a workaround):** `level` stays on the
record but is NOT listed in the projection's field set, and the proto says so at the site.
Fixing the generator's import emission is a one-line-ish tacplus change but it is a
generator change, which is out of Piece A's scope; it belongs with Piece B/C. After
dropping `level` from the projection, `cargo build` of `pull-task-activity` against the
regenerated tree finishes clean.

### 3. Q5 is already moot — the harness is not gameable the way the doc feared.

The design doc (§2 Piece A, §4 Q5) warns that Piece A alone would flip
`accounting.real-emitter` green "as soon as ListTopics mentions 'account'". That is no
longer true of the harness in the tree. With the topic live it reports:

```
FAIL(not-implemented) accounting.real-emitter
  accounting topic exists but no real service emits into it — topic declared, no emitter
  wired. eventbus lists "vservices.accounting-record.accountingrecord.AccountingRecordBus.Stream",
  but every SetEmitter caller under platform/ and testapp/ is generated chassis or a test
  fixture (11 generated, 0 fixture, 0 hand-written).
```

So the check distinguishes "topic exists" from "someone emits", and Piece A honestly does
not close item 14. Nothing was gamed and nothing needs tightening for that reason.

## Success criteria — actual results

### The proto compiles; the whole fleet regenerates

`testapp/build.sh --generate-only` with local proto sources, all 12 services:

```
==> generating with platform + app protos
    platform: /Users/jody/workspace/schematic/platform/proto
    app:      /Users/jody/workspace/schematic/testapp/proto
  OK   eventbus generated       OK   opa generated
  OK   gateway generated        OK   sentinel generated
  OK   graphdb generated        OK   stratum generated
  OK   kernel generated         OK   timedb generated
  OK   kmaker generated         OK   echo generated
  OK   oauth generated          OK   tasks generated
generation complete.
```

### eventbus lists an `accountingrecord.*` topic

Queried the live cluster the same way `testapp/e2e/run.sh:746-748` does (mint kernel SA
token → exchange for an eventbus-audience token → `grpcurl` the port-forwarded `eventbus0`
pod → `eventbus.Admin/ListTopics`).

Before (7 topics):

```
services.tasks.tasks.TaskActivityBus.Stream
services.tasks.tasks.TaskEventBus.TaskEvents
vservices.agent-sensor.agentsensor.AgentEventBus.EventStream
vservices.agent-sensor.trace.TraceEventBus.TraceEvents
vservices.gitlab-connector.gitlabconnector.GitLabConnectorEventBus.EventStream
vservices.llm-sensor.llmsensor.LlmEventBus.EventStream
vservices.scm-sensor.scmsensor.ScmEventBus.EventStream
```

After redeploying eventbus and stratum (8 topics):

```
services.tasks.tasks.TaskActivityBus.Stream
services.tasks.tasks.TaskEventBus.TaskEvents
vservices.accounting-record.accountingrecord.AccountingRecordBus.Stream   <-- new
vservices.agent-sensor.agentsensor.AgentEventBus.EventStream
vservices.agent-sensor.trace.TraceEventBus.TraceEvents
vservices.gitlab-connector.gitlabconnector.GitLabConnectorEventBus.EventStream
vservices.llm-sensor.llmsensor.LlmEventBus.EventStream
vservices.scm-sensor.scmsensor.ScmEventBus.EventStream
```

Mechanism note, since it differs from the design doc's assumption: eventbus's
`ListTopics` returns *Kafka's* topic list (`kafka/TopicManager.listTopics()`), and the only
caller of `ensureTopicExists` is `PublishHandler`. The topic therefore appears because
**stratum's generated subscriber connects to it at startup**, not because eventbus declares
it — confirmed in the stratum logs on all three replicas:

```
... AccountingrecordAccountingRecordBusStreamSubscriber subscribeOnce
INFO: accountingrecord.stream subscription stream started
```

### Stratum has a typed store + query surface for AccountingRecord

Generated into `platform/services/stratum/model/target/generated-sources/model/proto/services/stratum.proto`
— the full plane, structurally identical to `TaskActivity`'s (34 `AccountingRecord`
references vs 32 for `TaskActivity`):

```proto
message AccountingRecordFilter {
  optional string request_id = 1;          // <- the correlation key, EQ-selectable
  optional string source_id = 2;
  optional int64  timestamp_ms_from = 3;
  optional int64  timestamp_ms_to = 4;     // <- the RANGE pair
  optional string method = 5;
  optional string spiffe_id = 6;
  ...
}
rpc CreateAccountingRecordQuery(OpenAccountingRecordQueryRequest) returns (QueryHandle);
rpc GetAccountingRecordCount(QueryHandle) returns (RecordCount);
rpc GetAccountingRecordSample(SampleAccountingRecordRequest) returns (...);
rpc ListAccountingRecord(PullAccountingRecordRequest) returns (PullAccountingRecordResponse);
rpc DeleteAccountingRecordQuery(QueryHandle) returns (QueryStats);
rpc UpdateAccountingRecordQuery(RefineAccountingRecordRequest) returns (QueryStats);
rpc WatchAccountingRecord(QueryHandle) returns (stream AccountingRecordWatchEvent);
rpc SubscribeAccountingRecordCount(QueryHandle) returns (stream RecordCount);
```

plus the ingest side, `.../s2sm/subscribe/AccountingrecordAccountingRecordBusStreamSubscriber.java`,
and the synthesized topic wrapper `message AccountingRecordBus { AccountingRecord stream = 1; }`
(tagged `// tacplus:generated EventBus topic wrapper for AccountingRecord` — not hand-written,
per `tasks.proto:256-259`).

### Harness score — **23/32, zero regressions, identical red set to the baseline**

Final run, on a settled cluster with no other worker's harness running:

```
passed 23 / 32 {'PASS': 23, 'FAIL(not-implemented)': 8, 'FAIL(blocked)': 1}
(no CHANGED lines — every check has the same status it had before this change)
--- red ---
  FAIL(not-implemented) owner_default.read-yes-write-no
  FAIL(not-implemented) entitlement.wider-than-self
  FAIL(not-implemented) trace.events-end-to-end
  FAIL(not-implemented) accounting.real-emitter
  FAIL(not-implemented) accounting.correlates-to-row
  FAIL(not-implemented) graphdb.landing-query
  FAIL(not-implemented) query.trait
  FAIL(not-implemented) cli.traces
  FAIL(blocked)         cli.install-events
```

Same nine ids as the baseline, same statuses. `stratum.landing`, `stratum.query`,
`timedb.landing-query` and `build.testapp` are all green again. **Zero regressions, and
`accounting.real-emitter` correctly stays red** — Piece A declares the topic but wires no
emitter, and the harness (unlike what the design doc predicted) says exactly that.

The two intermediate 20/32 runs and their diagnosis are kept below, because the failure mode
is worth knowing about and is not this change's fault.

### Intermediate runs — the cluster is shared and was being changed under me

**Baseline, run before touching anything (my own run, not taken from `state.md`): 23/32 PASS**,
`FAIL(not-implemented)=8`, `FAIL(blocked)=1`. Red set:

```
owner_default.read-yes-write-no    entitlement.wider-than-self   trace.events-end-to-end
accounting.real-emitter            accounting.correlates-to-row  graphdb.landing-query
query.trait                        cli.traces                    cli.install-events (blocked)
```

(Note this differs slightly in *shape* from iteration 5162's recorded "7 not-implemented /
2 blocked" — same 23/32 and same nine ids, `cli.install-events` just classifies differently
now. Same red set.)

Two further runs after deploying: **20/32 both times**, with the same three checks moving:

| check | baseline | run 2 | run 3 |
|---|---|---|---|
| `build.testapp` | PASS | **FAIL(regression)** | PASS (fixed — see the `EventLevel` defect) |
| `stratum.landing` | PASS | FAIL(not-implemented) | FAIL(not-implemented) |
| `stratum.query` | PASS | FAIL(blocked) | FAIL(blocked) |
| `timedb.landing-query` | PASS | PASS | FAIL(not-implemented) |

`build.testapp` was mine and is fixed. **The stratum/timedb reds I could not attribute to
this change, and there is direct evidence they are environmental:**

1. **Another process is deploying to the same cluster.** I deployed stratum
   `sha256:416cb814…` at 18:51:56 (ControllerRevision 22) and confirmed the rollout. Some
   time after, the StatefulSet was set **back** to `sha256:066538ea…` — my *first* build —
   as ControllerRevision **23**. I did not do that. The StatefulSet is Helm-managed
   (`meta.helm.sh/release-name: schematic`) with no `managedFields` entry naming a
   controller, so this was an external `helm`/`kubectl` actor, not a reconciler I could
   find in-namespace (there is no `operator` pod running).
2. **A second `testapp/e2e/run.sh` was running concurrently**, visible as its own
   port-forwards on the harness's ports:
   `kubectl port-forward svc/{kernel-0,stratum-0,timedb0,eventbus0,gateway0} -n schematic 947{1..5}`
   — processes I did not start. Two harnesses on one cluster also share the
   `max_open_tasks` budget.
3. **Stratum pods were 30–90 seconds old with nothing of mine running** — the StatefulSet
   was being rolled continuously by that other actor. A stratum that restarts every few
   minutes can never hold a Kafka subscription: eventbus's log shows the `stratum` consumer
   group at **generation 189 and climbing**, with `TaskActivityBus` partitions assigned and
   revoked in the same second and
   `runLoop exited with exception on topic services.tasks.tasks.TaskActivityBus.Stream:
   CANCELLED: call already cancelled`. That is exactly the failure the reds describe, and it
   is the known signature in `memory/stratum-rollout-kafka-rebalance-false-regression.md`.

**What I can say positively:** `services.tasks.tasks.TaskActivityBus.Stream` has real
messages (`latestOffsets` summing to 354), and the new
`vservices.accounting-record.accountingrecord.AccountingRecordBus.Stream` sits at offset 0
on all 6 partitions — correct, since Piece A wires no emitter. Stratum's own logs show all
three replicas subscribing to `tasks.stream`, `agentsensor.event_stream` and the new
`accountingrecord.stream` cleanly at each start.

**Resolved:** the fourth run, taken after the other worker's harness exited and the stratum
pods had been up several minutes, is 23/32 with the baseline red set (above). The reds were
the churn, not the change. I also restarted `tasks0` once along the way (its EventBus
publisher had not reconnected after the eventbus pod was replaced — eventbus logged no
publishes for 30 minutes); that alone did not fix it, settling the cluster did.

**One thing I could not win, stated plainly:** something outside this session keeps setting
the stratum StatefulSet back to `stratum@sha256:066538ea…` (my *first* build) within minutes
of my setting `sha256:416cb814…`. I re-asserted the corrected digest twice and it was
reverted both times, so **the currently deployed stratum is `066538ea`, not `416cb814`**.
Functionally they differ only in whether `level` appears in the timeseries projection —
stratum's retention sweep keys off `timestamp_ms` either way and stratum stores full records,
so the 23/32 result stands for both. But if you want the deployed image to match the proto in
the tree, re-apply:

```
kubectl -n schematic set image sts/stratum \
  stratum=us-central1-docker.pkg.dev/jody1-328023/seabeam/schematic/stratum@sha256:416cb8140ff7ddd7d0396060f8cf12813448c23909c48b97e961923e62e78e33
```

and find whoever is re-pinning it first, or it will revert again.

## Build / deploy detail

- Generator: `tacplus` HEAD `db610bd8`. `testapp/build.sh`'s `TACPLUS_PINNED_SHA` is still
  `3e1df72b`, one commit BEHIND HEAD; `db610bd8` is a direct descendant (verified with
  `git merge-base --is-ancestor`) and is the commit iteration 5147 actually landed, so
  generating at it moves the fleet forward, never backwards. Used the documented one-off
  override `TACPLUS_PINNED_SHA=db610bd8...` rather than editing `build.sh`'s default or
  checking out an older tree. **The pin in `build.sh` should be advanced to `db610bd8` by
  whoever owns it — a lagging pin is the exact failure mode that file's own comment
  documents at length.** I did not edit it, as that is outside this unit.
- Proto sources overridden to local paths (`PLATFORM_PROTO=$HOME/workspace/schematic/platform/proto`,
  `APP_PROTO=$HOME/workspace/schematic/testapp/proto`) — required, since the defaults are
  GitLab URLs and the new proto is local and uncommitted. `protosrc.Resolve` uses a local
  directory as-is (working tree, no commit needed), `tacplus/pkg/protosrc/resolve.go:47`.
- Images built and pushed for eventbus and stratum only, then deployed **by digest** (not
  by the mutable `:local` tag):
  - `eventbus@sha256:3b759935f81930d66e6a1c9ead5bf1c0c9e81b082d428ed59addebb45e675d8b`
  - `stratum@sha256:416cb8140ff7ddd7d0396060f8cf12813448c23909c48b97e961923e62e78e33`
  - Previous digests, for rollback: eventbus `sha256:77e1cb1e663fb2197bbd5d76ba0524dfcfc1c9067fce093ce5299ef312859c50`,
    stratum `sha256:38473a4d50e64b06b1927aa96283143f3acc0a2fd62a06a3face14b074d38d91`.
  - Both rollouts completed; all 18 pods in `schematic` Ready afterwards.

## Known-stale, deliberately not rebuilt

**timedb.** Adding the `AccountingRow` timeseries projection changes timedb's generated
surface too, and timedb was regenerated but NOT rebuilt/redeployed (the task scoped the
rebuild to stratum + eventbus). This is currently harmless — nothing publishes to the
accounting topic until Piece B, so there is nothing for timedb to project — but **timedb
must be rebuilt and redeployed as part of Piece B**, before the first emitter is wired, or
accounting rows will land in stratum with no timeseries projection behind them.

The same is true in the weaker sense for every other platform service: they were
regenerated (so their trees now carry `accountingrecord` and vservice field_id 52) but not
reimaged. Nothing depends on that until something publishes.
