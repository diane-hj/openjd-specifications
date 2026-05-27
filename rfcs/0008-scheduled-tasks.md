
* Feature Name: Scheduled Tasks
* Author(s): (fill in)
* RFC Tracking Issue: (pending)
* Start Date: 2026-05-27
* Specification Version: 2023-09 extension SCHEDULE
* Accepted On: (pending)
* Depends On: RFC 0002 (model extensions)

## Summary

This RFC proposes a `schedule` field on Job Templates that turns a job into a recurring set
of *runs* driven by a cron expression and an end timestamp. Each step in the Job Template is
treated as a *step template*. On every scheduled trigger the scheduler instantiates each step
template — with the per-run values `{{Schedule.Run}}` and `{{Schedule.Timestamp}}` bound — and
appends the resulting steps to the job's step list. The job's step list therefore grows by
`len(steps)` on every trigger; each appended step is a normal OpenJD step with whatever task
parameter space its template defined. The job is complete after the schedule's end time and
all instantiated tasks have finished.

## Basic Examples

### Example 1 — Hourly health check

A simple job that runs an HTTP health check every hour from job submission until a fixed end
time. The job has one step template with no parameter space, so each trigger appends a new
single-task step `Probe-Run-N`.

```yaml
specificationVersion: 'jobtemplate-2023-09'
extensions:
- SCHEDULE
name: Hourly Health Check ({{Param.Endpoint}})
parameterDefinitions:
  - name: Endpoint
    type: STRING
    default: https://example.com/health
schedule:
  cron: "0 * * * *"            # top of every hour
  timezone: "UTC"
  endAt: "2026-06-30T00:00:00Z"
steps:
  # The name field is a format string under the SCHEDULE extension.
  # Schedule.Run is bound to a fresh integer (1, 2, 3, ...) on each trigger,
  # so successive instantiations are appended as Probe-Run-1, Probe-Run-2, ...
  - name: "Probe-Run-{{Schedule.Run}}"
    script:
      actions:
        onRun:
          command: bash
          args: ["{{Task.File.Probe}}"]
      embeddedFiles:
        - name: Probe
          filename: probe.sh
          type: TEXT
          data: |
            set -euo pipefail
            echo "Run #{{Schedule.Run}} triggered at {{Schedule.Timestamp}}"
            curl --fail --silent --show-error '{{Param.Endpoint}}'
```

After three triggers the job's live step list is:
`[Probe-Run-1, Probe-Run-2, Probe-Run-3]`. Each step has exactly one task.

### Example 2 — Nightly render of the latest scene

A nightly render at 23:00 PT. The Job Template has one step template with an internal
`Frame` task parameter space. Each trigger appends one new step `Render-Run-N` whose task
list is the full set of frames.

```yaml
specificationVersion: 'jobtemplate-2023-09'
extensions:
- SCHEDULE
name: Nightly Render — {{Param.SceneName}}
parameterDefinitions:
  - name: SceneFile
    type: PATH
    objectType: FILE
    dataFlow: IN
  - name: SceneName
    type: STRING
    default: shot_001
  - name: OutputDir
    type: PATH
    objectType: DIRECTORY
    dataFlow: OUT
  - name: Frames
    type: STRING
    default: "1-100"
schedule:
  cron: "0 23 * * *"             # 23:00 every day
  timezone: "America/Los_Angeles"
  startAt: "2026-06-01T00:00:00-07:00"
  endAt:   "2026-09-01T00:00:00-07:00"
steps:
  - name: "Render-Run-{{Schedule.Run}}"
    parameterSpace:
      taskParameterDefinitions:
        # The step's task space is the same for every run instance.
        # It does not need to reference Schedule.Run.
        - name: Frame
          type: INT
          range: "{{Param.Frames}}"
    script:
      actions:
        onRun:
          command: bash
          args: ["{{Task.File.Render}}"]
      embeddedFiles:
        - name: Render
          filename: render.sh
          type: TEXT
          data: |
            set -euo pipefail
            # Schedule.Timestamp is bound to the trigger time of this step instance.
            # Use the date portion as the per-run output subdirectory.
            TS='{{Schedule.Timestamp}}'
            RUN_DATE="${TS%%T*}"
            OUT='{{Param.OutputDir}}'/"$RUN_DATE"/frame_$(printf '%04d' {{Task.Param.Frame}}).exr
            mkdir -p "$(dirname "$OUT")"
            render -scenefile '{{Param.SceneFile}}' -frame {{Task.Param.Frame}} -o "$OUT"
```

After three triggers the live step list is `[Render-Run-1, Render-Run-2, Render-Run-3]`.
Each of those steps has 100 tasks (one per frame in the parameter space).

### Example 3 — ETL pipeline (Fetch → Transform → Publish) every 15 minutes

A three-stage pipeline with intra-run dependencies. The Job Template has three step
templates. Each trigger instantiates all three, wiring each instance's `dependsOn` to the
matching instance from the same run (also a format string under SCHEDULE).

```yaml
specificationVersion: 'jobtemplate-2023-09'
extensions:
- SCHEDULE
name: Hourly ETL — {{Param.Dataset}}
parameterDefinitions:
  - name: Dataset
    type: STRING
    default: orders
  - name: Workdir
    type: PATH
    objectType: DIRECTORY
    dataFlow: INOUT
  - name: PublishBucket
    type: STRING
    default: s3://my-warehouse/orders
schedule:
  cron: "*/15 9-17 * * 1-5"       # every 15 minutes, 9am-5pm, Mon-Fri
  timezone: "America/New_York"
  endAt: "2026-12-31T23:59:59-05:00"
steps:
  - name: "Fetch-Run-{{Schedule.Run}}"
    script:
      actions:
        onRun:
          command: bash
          args: ["{{Task.File.Fetch}}"]
      embeddedFiles:
        - name: Fetch
          filename: fetch.sh
          type: TEXT
          data: |
            set -euo pipefail
            STAGE='{{Param.Workdir}}/run_{{Schedule.Run}}/raw'
            mkdir -p "$STAGE"
            fetch-data --dataset '{{Param.Dataset}}' \
                       --since '{{Schedule.Timestamp}}' \
                       --out "$STAGE"
  - name: "Transform-Run-{{Schedule.Run}}"
    dependencies:
      # dependsOn is a format string under the SCHEDULE extension, so each instance
      # binds to the matching Fetch step from the same run.
      - dependsOn: "Fetch-Run-{{Schedule.Run}}"
    script:
      actions:
        onRun:
          command: bash
          args: ["{{Task.File.Transform}}"]
      embeddedFiles:
        - name: Transform
          filename: transform.sh
          type: TEXT
          data: |
            set -euo pipefail
            BASE='{{Param.Workdir}}/run_{{Schedule.Run}}'
            transform "$BASE/raw" "$BASE/curated"
  - name: "Publish-Run-{{Schedule.Run}}"
    dependencies:
      - dependsOn: "Transform-Run-{{Schedule.Run}}"
    script:
      actions:
        onRun:
          command: bash
          args: ["{{Task.File.Publish}}"]
      embeddedFiles:
        - name: Publish
          filename: publish.sh
          type: TEXT
          data: |
            set -euo pipefail
            BASE='{{Param.Workdir}}/run_{{Schedule.Run}}'
            aws s3 sync "$BASE/curated" '{{Param.PublishBucket}}'/run={{Schedule.Run}}/
```

After the first three triggers the live step list is, in order:
`[Fetch-Run-1, Transform-Run-1, Publish-Run-1, Fetch-Run-2, Transform-Run-2, Publish-Run-2, Fetch-Run-3, Transform-Run-3, Publish-Run-3]`.
Each `Run-N` group is wired together by step dependencies; the groups are independent of
each other and may overlap on the worker fleet.

## Motivation

Recurring work is a fundamental pattern in compute-heavy pipelines:

- **Time-driven content production.** A studio re-renders the latest scene file every night so
  the next morning's review session always has fresh dailies. A simulation house runs a weekly
  long-take to detect drift in the underlying assets.
- **Operational pipelines.** Data warehousing, log rollups, periodic re-indexing, and ML feature
  recomputation are commonly defined as "do X every N minutes". Today these pipelines live
  outside the OpenJD ecosystem because OpenJD has no first-class concept of a recurring job.
- **Calibration and monitoring.** Periodic health checks, golden-image renders, and benchmark
  suites need to run on a schedule and produce a comparable artifact per run.

Today users have to choose between two unsatisfying options:

1. Submit a fresh job for each run via an external scheduler (cron, Airflow, EventBridge,
   etc.). This creates an explosion of single-shot jobs in the render management system,
   loses cross-run identity, and forces the user to re-derive what should be a single job's
   lineage from outside metadata.

2. Hard-code a long, finite parameter space at submission time, e.g. "1..720 hours" for a
   month of hourly runs. This locks in the schedule when the job is created, requires
   re-submission on every change, and ignores cron's expressiveness.

This proposal treats "scheduled" as a property of the job. A single job ID accumulates the
full history of runs; each run is a clean, independent group of steps that operators can
inspect, retry, or cancel as a unit.

The choice to grow the *step list* per trigger (rather than grow the task list inside fixed
steps) keeps each scheduled run self-contained:

- A run's full pipeline shape is visible as a contiguous group of steps in the job. Operators
  can answer "show me run 5" with a list operation, and "retry just run 5" by retrying that
  group of steps.
- Each step instance is a normal OpenJD step. The internal task parameter space, host
  requirements, environments, and step dependencies all behave exactly as they do in
  non-scheduled jobs.
- Cross-step dependencies are run-local by default — `Transform-Run-3` depends on
  `Fetch-Run-3`, not on any other run — which matches operator intuition and avoids the
  pitfalls of merging unrelated runs into a single dependency graph.

## Specification

> Changes to [the template schema](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas).

### Job Template root element

> A modification to [`Job Template — Root Elements`](../wiki/2023-09-Template-Schemas#11-Job-Template).

```diff
  specificationVersion: "jobtemplate-2023-09"
  $schema: <string> # @optional
  extensions: [ <ExtensionName>, ... ] # @optional
  name: <JobName> # @fmtstring
  description: <Description> # @optional
  parameterDefinitions:  [ <JobParameterDefinition>, ... ] # @optional
+ schedule: <Schedule> # @optional @extension SCHEDULE
  jobEnvironments: [ <Environment>, ... ] # @optional
  steps: [<StepTemplate>, ...]
```

> Add the following item to the description of root elements, after *parameterDefinitions*:

```diff
+ N. *schedule* — If provided, declares the Job as a recurring scheduled Job. Requires the
+    `SCHEDULE` extension. Every entry in *steps* is treated as a step template that is
+    instantiated and appended to the live step list on each scheduled trigger. See:
+    [<Schedule>](#13-schedule) and [Step instantiation](#741-scheduled-job-lifecycle-extension-schedule).
```

### New section: `<Schedule>`

> A new section after [section 1.2.1](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#121-merging-environment-template-parameter-definitions).

```diff
+ ### 1.3. `<Schedule>` `@extension SCHEDULE`
+
+ Declares a Job as a recurring scheduled Job. While the schedule is active, the scheduler
+ periodically fires triggers according to the cron expression. On each trigger, every entry
+ in the Job Template's *steps* list is instantiated with `Schedule.Run` and
+ `Schedule.Timestamp` bound to the trigger's run number and time, and the resulting steps
+ are appended to the Job's live step list.
+
+ A `<Schedule>` is the object:
+
+ ```yaml
+ cron: <CronExpression>
+ timezone: <IANATimezone> # @optional
+ startAt: <Timestamp>     # @optional
+ endAt: <Timestamp>
+ catchup: enum("NONE", "LATEST", "ALL") # @optional
+ maxRuns: <integer>       # @optional
+ ```
+
+ Where:
+
+ 1. *cron* — A standard 5-field cron expression: `<minute> <hour> <day-of-month> <month> <day-of-week>`.
+    See [`<CronExpression>`](#131-cronexpression).
+ 2. *timezone* — An IANA Time Zone Database name (e.g. `"America/Los_Angeles"`). Default `"UTC"`.
+ 3. *startAt* — Earliest trigger time as an ISO 8601 timestamp. Default: Job creation time.
+ 4. *endAt* — Latest trigger time as an ISO 8601 timestamp. *endAt* must be strictly greater
+    than *startAt*. After *endAt* and after all triggered Tasks have completed, the Job
+    transitions to a terminal state.
+ 5. *catchup* — How to treat triggers that elapsed while the schedule was unavailable
+    (scheduler outage, paused Job, etc.). Default `"NONE"`.
+    1. `"NONE"` — Missed triggers are dropped.
+    2. `"LATEST"` — At most one missed trigger is fired.
+    3. `"ALL"` — Every missed trigger is fired in order.
+ 6. *maxRuns* — Optional absolute upper bound on the number of triggers. Once
+    `Schedule.RunCount == maxRuns`, no further triggers are produced.
+    1. Minimum value: 1
+
+ #### 1.3.1. `<CronExpression>`
+
+ A string containing a 5-field cron expression: `<minute> <hour> <day-of-month> <month> <day-of-week>`,
+ supporting numeric values, `*`, `a-b` ranges, `a,b,c` lists, and `*/n` or `a-b/n` step values.
+
+ Examples: `"0 * * * *"`, `"*/15 9-17 * * 1-5"`, `"0 23 * * *"`.
+
+ #### 1.3.2. `<Timestamp>`
+
+ A string containing an ISO 8601 timestamp. If no UTC offset is provided, the *timezone* of
+ the enclosing `<Schedule>` is applied.
+
+ #### 1.3.3. `<IANATimezone>`
+
+ A string naming a zone in the IANA Time Zone Database. Schedulers without IANA database
+ access may accept only `"UTC"`.
```

### Step name and dependency become Format Strings

> A modification to [section 3.1. `<StepName>`](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#31-stepname).

```diff
- A string subject to the following constraints:
+ A string subject to the following constraints. When the `SCHEDULE` extension is enabled,
+ this is a [Format String](#73-format-strings) instead of a literal string, resolved at
+ Job creation time for non-scheduled Jobs and at Schedule trigger time for scheduled Jobs.

  1. Allowed characters: Any unicode character except those in the Cc unicode character category.
  2. Minimum length: 1 character.
  3. Maximum length: 64 characters. 512 characters if using the `FEATURE_BUNDLE_1` extension.
+ 4. Constraint specific to scheduled Jobs: when the Job Template defines a `schedule`, the
+    *name* field of every entry in *steps* must produce a name that varies with
+    `Schedule.Run`. The simplest way to satisfy this constraint is to embed
+    `{{Schedule.Run}}` in the format string.
```

> A modification to [section 3.2. `<StepDependency>`](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#32-stepdependency).

```diff
  A `<StepDependency>` is the object:

  ```yaml
- dependsOn: "<StepName>"
+ dependsOn: "<StepName>" # @fmtstring under @extension SCHEDULE
  ```

  Where:

  1. *dependsOn* — Provides the name of another step in the same job upon which this step
     depends. The Task(s) of this Step may be scheduled only when the depended-upon Step has
     fully completed successfully.
+    Under the `SCHEDULE` extension, *dependsOn* is a Format String resolved at Schedule
+    trigger time. The resolved value must match the resolved name of another step in the
+    same Job. Cross-run dependencies (referencing a step from a different `Schedule.Run`)
+    are out of scope for this RFC; see [Open Questions](#open-questions).
```

### New built-in template values

> Add the following rows to the table in [section 7.3.1. `Value References`](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#731-value-references).

```diff
+ |`Schedule.Run`| `@extension SCHEDULE`. The 1-indexed run number of the current step instance. This is an `int`. The first triggered run is 1, the second is 2, and so on. | Available in every Format String within a `<StepTemplate>` (including its *name*, *dependencies*, *parameterSpace*, *script*, and *stepEnvironments*) of a scheduled Job. |
+ |`Schedule.Timestamp`| `@extension SCHEDULE`. The ISO 8601 timestamp at which the current step instance's run was triggered, in the schedule's timezone. This is a `string`. | Same scope as `Schedule.Run`. |
+ |`Schedule.RunCount`| `@extension SCHEDULE`. The total number of runs that have been triggered so far at the time this step instance is created. This is an `int`. For the most recently triggered run, `Schedule.RunCount == Schedule.Run`. | Same scope as `Schedule.Run`. |
+ |`Schedule.Cron`| `@extension SCHEDULE`. The cron expression from the Job Template's *schedule*. This is a `string`. | Available in every Format String in the Job Template. |
+ |`Schedule.Timezone`| `@extension SCHEDULE`. The IANA timezone name from the Job Template's *schedule*. This is a `string`. | Available in every Format String in the Job Template. |
+ |`Schedule.StartAt`| `@extension SCHEDULE`. The resolved *startAt* timestamp as an ISO 8601 string. | Available in every Format String in the Job Template. |
+ |`Schedule.EndAt`| `@extension SCHEDULE`. The resolved *endAt* timestamp as an ISO 8601 string. | Available in every Format String in the Job Template. |
```

### Template processing stages

> A modification to the table in [section 7.4. `Template Processing Stages`](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#74-template-processing-stages).
> Add a new row between **Job creation** and **Task execution**:

```diff
+ | **Schedule trigger** `@extension SCHEDULE` | A scheduled trigger fires (cron match within `[startAt, endAt]`) | All values from job creation, plus `Schedule.Run`, `Schedule.Timestamp`, `Schedule.RunCount` for the new run | `Task.Param.*`, `Session.*`, `Task.File.*`, `Env.File.*` | Each entry in *steps* is instantiated with the new run's values bound. The resolved step instances are appended to the Job's live step list. Format strings inside step templates that reference `Schedule.Run` or `Schedule.Timestamp` are resolved here; format strings already resolved at Job creation are not re-evaluated. |
```

> Add a new subsection:

```diff
+ #### 7.4.1. Scheduled Job lifecycle `@extension SCHEDULE`
+
+ A scheduled Job is created in a `WAITING` state. Its live step list is initially empty.
+ The scheduler transitions the Job through the following states:
+
+ - `WAITING` — Created, no triggers have fired yet. Steps: 0.
+ - `RUNNING` — At least one trigger has fired. Each trigger appends `len(template.steps)`
+   new step instances to the live step list, in their template order.
+ - `COMPLETING` — Either *endAt* has passed, *maxRuns* has been reached, or the Job has
+   been cancelled. No further triggers will be produced. Already-instantiated steps and
+   their tasks may still be running.
+ - Terminal (`SUCCEEDED` / `FAILED` / `CANCELLED`) — Reached when in `COMPLETING` and all
+   instantiated Tasks have reached a terminal state.
+
+ Step instantiation is purely additive: a step instance, once appended to the live step
+ list, is never modified. The values of `Schedule.Run` and `Schedule.Timestamp` are
+ captured at the moment of instantiation and remain constant for the life of that step
+ instance and its tasks.
+
+ The order of step instances in the live step list is `<run-1 step templates in order>,
+ <run-2 step templates in order>, ...`. Step instances within the same run can declare
+ dependencies on each other through *dependsOn* format strings that resolve to other
+ same-run step names; cross-run dependencies are not supported in this RFC.
```

## Design Choice Rationale

### Growing the step list (instead of growing task lists inside fixed steps)

An alternative design considered for this RFC has the Job's step list fixed at submission
and the task list inside each step grow on every trigger (e.g. each step has a "Run" task
parameter bound to a dynamic `Schedule.RunRange`). That approach keeps the job structure
static and constrains the dynamic dimension to a single integer per step. The chosen
approach instead grows the step list. It was preferred because:

- **Each run is a self-contained pipeline.** A run is a contiguous group of steps that
  operators can inspect, retry, cancel, or visualize as a unit. The "task list grows"
  alternative interleaves the per-run state inside every step, making "show me run 5"
  harder to express.
- **Step dependencies stay simple.** In the chosen design, `Transform-Run-3` depends on
  `Fetch-Run-3` through standard OpenJD step dependencies. In the alternative, step-level
  dependencies still mean "all of A before any of B", and run-level alignment has to be
  achieved through task-parameter cross products that mix run-N and run-M tasks across
  steps.
- **No new parameter-space concept.** Each instantiated step is a normal OpenJD step. The
  spec does not need to introduce a "growing task parameter space" or a new dynamic
  `list[int]` value. The only new thing is the run-time step instantiation step.
- **Pipeline shape can vary in future RFCs.** Run-N being its own group of steps makes it
  easy to imagine future extensions that, e.g., conditionally include or skip step
  templates per run, without disturbing the structure of other runs.

### Step names and `dependsOn` become format strings under SCHEDULE

For the scheduler to instantiate steps with `Schedule.Run` substituted in, both the step
*name* and the *dependsOn* of any step dependency must be format strings. This is a
backward-compatible promotion: existing static names continue to resolve to themselves.
The RFC limits this promotion to the SCHEDULE extension to keep the change minimal.

### `Schedule.Run` is a per-instance scalar (not a list)

Each step instance is a fresh, immutable copy of a step template, so `Schedule.Run` is just
the run number for that copy. There is no need for a `list[int]` value, and step instances
do not need to know about each other's run numbers. Operators who want to compare across
runs use the job's step list directly.

### Cron only

A 5-field cron expression covers the vast majority of recurring schedules. The `<Schedule>`
object is a flat record rather than a `type:`-tagged union so v1 stays small. A future RFC
can introduce alternative trigger types (interval-only, event-driven, calendar-window) by
adding sibling fields.

### `endAt` is required, `startAt` is optional

Every scheduled job is bounded. Requiring *endAt* prevents the common operational mistake
of submitting a job that runs forever and accumulates steps indefinitely.

### `catchup` defaults to `NONE`

A scheduler outage is the most common reason for a missed trigger. Replaying missed
triggers often does the wrong thing (e.g., re-running expired health probes, double-
publishing data). Users who explicitly want backfill can opt in to `LATEST` or `ALL`.

### Run numbers start at 1

Run numbers are user-visible (in step names, output paths, log lines). Starting at 1
matches how humans count and how most cron-driven systems present run counts.

### Step instantiation is purely additive

Once a step instance is appended to the live step list, its definition is never modified.
This preserves the existing OpenJD invariant that a Job's structure is stable from a
worker's point of view: a worker host that already accepted `Render-Run-3`'s tasks can
keep working on them regardless of subsequent triggers, and the schedule trigger logic
never has to reconcile partially-running steps with new template values.

## Prior Art

- **Apache Airflow.** Airflow's DAG schedule produces a fresh "DAG run" on each trigger,
  which is exactly the per-run group of step instances this RFC produces. Airflow exposes
  per-run context as `{{ ds }}` and `{{ execution_date }}`; `Schedule.Timestamp` plays the
  same role here.
- **Kubernetes CronJob.** Each trigger creates a fresh Job object. The "growing job"
  pattern in this RFC differs by keeping all runs under one OpenJD Job ID for cross-run
  observability, but the per-run instantiation model is similar.
- **AWS EventBridge Scheduler / cron / systemd timers.** All confirm cron + timezone +
  end-time as the standard schedule shape.
- **OpenCue.** OpenCue does not natively support recurring jobs; users wrap submission in
  external schedulers. Conversations with OpenCue users (see GitHub Discussions) indicate
  this is a frequent pain point.

## Implementation Impact

### Python (openjd-model, openjd-sessions, openjd-cli)

- **openjd-model.**
  - New `Schedule` Pydantic model on the Job Template root, gated by the `SCHEDULE` extension.
  - Step `name` and `<StepDependency>.dependsOn` become format strings when SCHEDULE is enabled.
  - Validation:
    - Cron expression parsing (e.g. `croniter`).
    - Timezone validation against `zoneinfo`.
    - Cross-validation between *startAt* and *endAt*.
    - Per-step constraint that the resolved name varies with `Schedule.Run` (the simplest
      form of this is "the format string contains a `{{Schedule.Run}}` reference"; a
      stricter check evaluates a few sample run numbers to confirm distinctness).
  - A new template-resolution mode "schedule trigger" that takes a step template and a run
    binding and produces a resolved step. The existing job-creation resolver remains
    responsible for everything else.
  - Estimated effort: medium.
- **openjd-sessions.** Unaffected. Sessions run a single instantiated step's tasks at a
  time, and step instances look like any other step.
- **openjd-cli.** A `--simulate-runs N` flag (or similar) for `openjd run`/`openjd check`
  that instantiates the first N runs locally so users can validate their templates without
  a live scheduler.

### Rust (openjd-rs)

- **openjd-model (Rust).** Same shape of change. Rust requires picking cron and timezone
  crates (`cron` + `chrono-tz`). Estimated effort: medium.
- **openjd-sessions (Rust).** Unaffected.
- **openjd-cli (Rust).** Same `--simulate-runs` flag as the Python CLI for parity.

### Performance implications

On each trigger the scheduler performs `O(steps)` template resolutions. Each resolution
is a small string-substitution pass, well under the cost of a normal job submission.
There is no impact on per-task hot paths.

### Are any features harder in one language than the other?

No. Both ecosystems have mature cron and timezone libraries, and the substitution model
is straightforward in both languages.

## Rejected Ideas

### Growing the task list inside a fixed step list

Considered making the Job's step list static and growing each step's task parameter space
on every trigger via a new dynamic `Schedule.RunRange : list[int]` value. Pros: the Job's
structure stays static. Cons: covered in [Design Choice Rationale](#growing-the-step-list-instead-of-growing-task-lists-inside-fixed-steps)
above. The per-run-as-a-step-group model in this RFC fits operator workflows ("rerun run
3", "show me runs 5–10") more naturally and avoids introducing a new dynamic list value
that can only be used in one specific spot in the schema.

### `type:`-tagged polymorphic schedules

Considered making `<Schedule>` a tagged union (`type: CRON`, `type: INTERVAL`, ...) so
that future schedule kinds can slot in without a schema migration. Rejected for v1 to
keep the surface area small. A future RFC can introduce a `type` field with `CRON` as
the default, preserving backward compatibility.

### Auto-generated run suffix on step names

Considered having the scheduler implicitly append `-Run-N` to every step name so the user
does not have to include `{{Schedule.Run}}`. Rejected because it violates OpenJD's
"explicit format strings" pattern, hides the dependency-resolution behaviour from the
template author, and forces a naming convention that may conflict with the user's own
naming choices.

### Per-step `schedule` declarations

Considered allowing `schedule` to be set on individual steps so that a Job could mix
"runs once" steps with "runs on schedule" steps. Rejected because the cross-step
semantics (does a once-step block? run at every trigger? run at the first trigger?) have
no obviously-right answer. The most common shape — every step is per-run, aligned by
`Schedule.Run` — is well-served by the simpler design. A targeted follow-up can introduce
a step-level `oncePerJob: true` flag if real workloads need it.

## Open Questions

### Once-per-job steps

Real pipelines often have a "warm-up" step (publish a manifest, prepare a destination)
and a "tear-down" step (notify, archive) that should run once for the whole scheduled
job. The current design forces every step to be per-run. Options:

1. Add a `schedule: ONCE` flag on a step.
2. Add a separate top-level `setupSteps:` / `teardownSteps:` list.
3. Defer to a follow-up RFC.

### Cross-run step dependencies

The current design supports only intra-run dependencies. Real workloads sometimes need a
serial chain across runs (e.g., "this run's reconciliation must wait for last run's
publish to finish"). A future RFC could allow `dependsOn: "Publish-Run-{{Schedule.Run - 1}}"`
with an `optional: true` flag (or similar) that elides the dependency when the resolved
name does not exist (i.e., on run 1). This requires the EXPR extension for arithmetic
in format strings.

### Concurrency

If run N+1 fires before run N has finished, the current design just lets both run
concurrently — they are independent groups of steps. Some workloads need the opposite
("one run at a time"). A `concurrencyPolicy: ALLOW | SKIP | QUEUE` field on `<Schedule>`
should be considered before final comments.

### Pause / resume

Render management systems commonly let an operator pause a job. For a scheduled job,
"pause" most likely means "stop firing triggers until I resume", and the *catchup* mode
then determines what happens to triggers that elapsed during the pause. We should confirm
that interpretation before final comments.

### Modifying the schedule mid-job

Should `endAt` be extendable after submission? Should `cron` be editable? Both have
legitimate use cases (extending a successful run, adjusting cadence after seeing load)
but require careful thought about whether step instances already produced are affected.
Out of scope for v1.

### Live step list cardinality

A long-running schedule can produce a very large live step list (e.g., a 15-minute
schedule for a year is ~35k runs × N step templates). Implementations may need to page,
archive, or cap the visible live step list. We should probably define a recommended
upper bound and behaviour when *maxRuns* or this implicit cap is reached.

## Copyright

This document is placed in the public domain or under the CC0-1.0-Universal license, whichever is more permissive.
