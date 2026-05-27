
* Feature Name: Scheduled Tasks
* Author(s): (fill in)
* RFC Tracking Issue: (pending)
* Start Date: 2026-05-27
* Specification Version: 2023-09 extension SCHEDULE
* Accepted On: (pending)
* Depends On: RFC 0002 (model extensions), RFC 0007 (extended parameter types — for `list[int]` and `list[string]`)

## Summary

This RFC proposes a `schedule` field on Job Templates that turns a job into a recurring,
append-only set of tasks driven by a cron expression and an end timestamp. While the schedule
is active, each scheduled trigger appends a new run number to the built-in `Schedule.RunRange`
list. Steps bind one of their task parameters to `Schedule.RunRange`, so each trigger grows
the step's parameter space by exactly one task per run. The job is complete after the
scheduled end time and all triggered tasks have finished. This is the first extension to
introduce a parameter space that grows over time, so an additional template-processing stage
("schedule trigger") is defined for resolving the dynamic ranges.

## Basic Examples

### Example 1 — Hourly health check

A simple job that runs an HTTP health check every hour from job submission until a fixed end
time. The schedule appends one run number per trigger; the single step renders that into one
new task per hour.

```yaml
specificationVersion: 'jobtemplate-2023-09'
extensions:
- SCHEDULE
- EXPR
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
  - name: Probe
    parameterSpace:
      taskParameterDefinitions:
        - name: Run
          type: INT
          # Built-in dynamic value: list[int] of the run numbers that have triggered so far.
          # The scheduler grows this list by one entry on each cron firing.
          range: "{{Schedule.RunRange}}"
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
            # Schedule.RunTimestamps is parallel to Schedule.RunRange; the EXPR extension
            # makes list indexing available so a task can find its own trigger time.
            TRIGGERED_AT='{{Schedule.RunTimestamps[Task.Param.Run - 1]}}'
            echo "Run #{{Task.Param.Run}} triggered at $TRIGGERED_AT"
            curl --fail --silent --show-error '{{Param.Endpoint}}'
```

### Example 2 — Nightly render of the latest scene

A nightly render at 23:00 PT. Each run renders the same set of frames, so the step has a
two-dimensional parameter space: `Run` (from the schedule) crossed with `Frame`. The
scheduler appends one row of `len(Frames) × 1` new tasks per trigger.

```yaml
specificationVersion: 'jobtemplate-2023-09'
extensions:
- SCHEDULE
- EXPR
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
    type: RANGE_EXPR
    default: "1-100"
schedule:
  cron: "0 23 * * *"               # 23:00 every day
  timezone: "America/Los_Angeles"
  startAt: "2026-06-01T00:00:00-07:00"
  endAt:   "2026-09-01T00:00:00-07:00"
steps:
  - name: Render
    parameterSpace:
      taskParameterDefinitions:
        - name: Run
          type: INT
          range: "{{Schedule.RunRange}}"
        - name: Frame
          type: INT
          range: "{{Param.Frames}}"
      # Default combination is the cross product, so each trigger appends |Frames| tasks.
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
            # Use the trigger date as the output subdirectory so each nightly run is
            # written to its own folder.
            TRIGGERED_AT='{{Schedule.RunTimestamps[Task.Param.Run - 1]}}'
            RUN_DATE="${TRIGGERED_AT%%T*}"
            OUT='{{Param.OutputDir}}'/"$RUN_DATE"/frame_$(printf '%04d' {{Task.Param.Frame}}).exr
            mkdir -p "$(dirname "$OUT")"
            render -scenefile '{{Param.SceneFile}}' -frame {{Task.Param.Frame}} -o "$OUT"
```

### Example 3 — Hourly ETL pipeline with step dependencies

A three-stage pipeline (Fetch → Transform → Publish) that runs every 15 minutes during
business hours. Each step references `Schedule.RunRange`, so each trigger appends one task
to each step. Step dependencies wire the pipeline together, and shared output paths are
keyed by `Run` so concurrent runs do not collide.

```yaml
specificationVersion: 'jobtemplate-2023-09'
extensions:
- SCHEDULE
- EXPR
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
  - name: Fetch
    parameterSpace:
      taskParameterDefinitions:
        - name: Run
          type: INT
          range: "{{Schedule.RunRange}}"
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
            STAGE='{{Param.Workdir}}/run_{{Task.Param.Run}}/raw'
            mkdir -p "$STAGE"
            fetch-data --dataset '{{Param.Dataset}}' \
                       --since '{{Schedule.RunTimestamps[Task.Param.Run - 1]}}' \
                       --out "$STAGE"
  - name: Transform
    dependencies:
      - dependsOn: Fetch
    parameterSpace:
      taskParameterDefinitions:
        - name: Run
          type: INT
          range: "{{Schedule.RunRange}}"
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
            BASE='{{Param.Workdir}}/run_{{Task.Param.Run}}'
            transform "$BASE/raw" "$BASE/curated"
  - name: Publish
    dependencies:
      - dependsOn: Transform
    parameterSpace:
      taskParameterDefinitions:
        - name: Run
          type: INT
          range: "{{Schedule.RunRange}}"
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
            BASE='{{Param.Workdir}}/run_{{Task.Param.Run}}'
            aws s3 sync "$BASE/curated" '{{Param.PublishBucket}}'/run={{Task.Param.Run}}/
```

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
   loses cross-run identity (no easy "show me runs 1–30 of this pipeline"), and forces the
   user to re-derive what should be a single job's lineage from outside metadata.

2. Hard-code a long, finite parameter space at submission time, e.g. "1..720 hours" for a
   month of hourly runs. This locks in the schedule when the job is created, requires
   re-submission on every change, leaves trailing tasks if the user wants to stop early,
   and ignores cron's expressiveness (e.g. "every 15 minutes during business hours, weekdays
   only").

Treating "scheduled" as a property of the job rather than a property of the surrounding
system has several benefits:

- A single job ID accumulates the full history of runs.
- The same template runs end-to-end and against the same parameters every cycle, with no
  external orchestrator getting out of sync.
- Tasks within a run can build on each other through normal step dependencies, instead of
  being forced into separate jobs.

The append-only `Schedule.RunRange` design keeps the change small. Steps express their
recurring work through the same task-parameter mechanism they already use; the only new
concept is a built-in dynamic list value that grows over time. Existing OpenJD features —
task chunking, host requirements, environments, path mapping, the EXPR extension — all
compose naturally with `Run` like any other task parameter.

## Specification

> Changes to [the template schema](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas).

### Job Template root element

> A modification to [`Job Template — Root Elements`](../wiki/2023-09-Template-Schemas#11-Job-Template)

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

> Add the following item to the description of root elements, immediately after *parameterDefinitions*:

```diff
+ N. *schedule* — If provided, declares the Job as a recurring scheduled Job. Requires the
+    `SCHEDULE` extension. While the Job is active, each scheduled trigger appends one new
+    run number to the built-in dynamic value `Schedule.RunRange`, and the scheduler grows
+    each Step's parameter space accordingly. See: <Schedule>.
```

### New section: `<Schedule>`

> A new section after [section 1.2.1. `Merging Environment Template Parameter Definitions`](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#121-merging-environment-template-parameter-definitions).

```diff
+ ### 1.3. `<Schedule>` `@extension SCHEDULE`
+
+ Declares a Job as a recurring scheduled Job. While the schedule is active, the scheduler
+ periodically fires triggers according to the cron expression. Each trigger appends a new
+ entry to the dynamic built-in values `Schedule.RunRange` and `Schedule.RunTimestamps` and
+ re-resolves any task parameter ranges that reference them, growing the parameter space of
+ each Step by one task per trigger.
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
+ 2. *timezone* — An IANA Time Zone Database name (e.g. `"America/Los_Angeles"`) used to
+    interpret the cron expression and `startAt`/`endAt` values that are not given as
+    explicit offsets. The default is `"UTC"`.
+ 3. *startAt* — Earliest time at which the schedule will fire a trigger, in ISO 8601
+    timestamp format. If omitted, the default is the time the Job was created. Triggers
+    that would fire before *startAt* are not produced.
+ 4. *endAt* — Latest time at which the schedule will fire a trigger, in ISO 8601 timestamp
+    format. After *endAt* has passed and all already-triggered Tasks have completed, the
+    Job transitions to a terminal state. *endAt* must be strictly greater than *startAt*.
+ 5. *catchup* — How the scheduler should treat trigger times that elapsed while the
+    schedule was unavailable (e.g. while the scheduler was offline, or the Job was paused).
+    The default is `"NONE"`.
+    1. `"NONE"` — Missed triggers are dropped.
+    2. `"LATEST"` — At most one missed trigger is fired, regardless of how many were missed.
+    3. `"ALL"` — Every missed trigger is fired in order.
+ 6. *maxRuns* — If provided, an absolute upper bound on the number of triggers the
+    schedule will produce. Once `len(Schedule.RunRange)` reaches *maxRuns*, no further
+    triggers are produced even if *endAt* has not been reached.
+    1. Minimum value: 1
+
+ #### 1.3.1. `<CronExpression>`
+
+ A string containing a 5-field cron expression in the form
+ `<minute> <hour> <day-of-month> <month> <day-of-week>`, with the standard syntax for
+ each field:
+
+ - Numeric values within the field's natural range
+ - `*` to match any value
+ - `a-b` for inclusive ranges
+ - `a,b,c` for explicit lists
+ - `*/n` or `a-b/n` for step values
+
+ Examples: `"0 * * * *"` (top of every hour), `"*/15 9-17 * * 1-5"` (every 15 minutes,
+ 9am-5pm, Mon-Fri), `"0 23 * * *"` (23:00 every day).
+
+ #### 1.3.2. `<Timestamp>`
+
+ A string containing an ISO 8601 timestamp. If the timestamp does not include a UTC offset,
+ it is interpreted in the *timezone* of the enclosing `<Schedule>`.
+
+ #### 1.3.3. `<IANATimezone>`
+
+ A string naming a zone in the IANA Time Zone Database, e.g. `"UTC"`, `"America/Los_Angeles"`,
+ `"Europe/Berlin"`. Schedulers that do not have access to the IANA Time Zone Database may
+ accept only `"UTC"`.
```

### Step parameter space requirement

> A modification to [`<StepParameterSpaceDefinition>`](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#34-stepparameterspacedefinition).
> Add the following constraint to the list of constraints on *taskParameterDefinitions*:

```diff
+ N. When the `SCHEDULE` extension is enabled and the Job Template defines a *schedule*,
+    each Step's *taskParameterDefinitions* list must contain exactly one
+    `<IntTaskParameterDefinition>` whose *range* property is the format string
+    `"{{Schedule.RunRange}}"` (or, with the `EXPR` extension enabled, an expression that
+    evaluates to `Schedule.RunRange`). This task parameter must not be of type `CHUNK[INT]`.
+    The Step's run-N task is the slice of its parameter space where this parameter equals N.
```

### New built-in template values

> Add the following rows to the table in [section 7.3.1. `Value References`](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#731-value-references):

```diff
+ |`Schedule.RunRange`| `@extension SCHEDULE`. The list of run numbers that have been triggered so far. This is a `list[int]` value. Without the `EXPR` extension, the value resolves as the string representation of an integer range expression (`<IntRangeExpr>`). On each scheduled trigger, the scheduler appends `len(Schedule.RunRange) + 1` to the list. | Available only as the *range* property of an `<IntTaskParameterDefinition>`. |
+ |`Schedule.RunTimestamps`| `@extension SCHEDULE`. A list parallel to `Schedule.RunRange` containing the ISO 8601 timestamp of each trigger, in the schedule's timezone. This is a `list[string]` value. The element at index `i` is the trigger time of run number `Schedule.RunRange[i]`. | Available within the Step Script Actions and Embedded Files. Requires the `EXPR` extension to subscript. |
+ |`Schedule.Cron`| `@extension SCHEDULE`. The cron expression from the Job Template's *schedule*. This is a `string`. | Available in every Format String in the Job Template. |
+ |`Schedule.Timezone`| `@extension SCHEDULE`. The IANA timezone name from the Job Template's *schedule*. This is a `string`. | Available in every Format String in the Job Template. |
+ |`Schedule.StartAt`| `@extension SCHEDULE`. The resolved *startAt* timestamp as an ISO 8601 string. This is a `string`. | Available in every Format String in the Job Template. |
+ |`Schedule.EndAt`| `@extension SCHEDULE`. The resolved *endAt* timestamp as an ISO 8601 string. This is a `string`. | Available in every Format String in the Job Template. |
```

### Template processing stages

> A modification to the table in [section 7.4. `Template Processing Stages`](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#74-template-processing-stages).
> Add a new row between **Job creation** and **Task execution**:

```diff
+ | **Schedule trigger** `@extension SCHEDULE` | A scheduled trigger fires (cron match within `[startAt, endAt]`) | All values from job creation, plus updated `Schedule.RunRange` and `Schedule.RunTimestamps` | `Task.Param.*`, `Session.*`, `Task.File.*`, `Env.File.*` | The scheduler appends a new run number and trigger timestamp, then re-resolves every task parameter *range* in every Step. New tasks added to a Step are exactly those tasks whose `Run` value equals the new run number. Format strings outside of task parameter ranges are not re-resolved. |
```

> Add a paragraph after the table:

```diff
+ The **Schedule trigger** stage applies only to Jobs whose Template uses the `SCHEDULE`
+ extension. The dynamic built-ins `Schedule.RunRange` and `Schedule.RunTimestamps` may
+ only be referenced in the *range* property of an `<IntTaskParameterDefinition>` and in
+ Step Script contexts respectively, because those are the only contexts that the scheduler
+ re-resolves on each trigger. References elsewhere are validation errors.
```

### Job lifecycle

> A new subsection in [section 7.4](https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#74-template-processing-stages):

```diff
+ #### 7.4.1. Scheduled Job lifecycle (`@extension SCHEDULE`)
+
+ A scheduled Job is created in a `WAITING` state. `Schedule.RunRange` is initially the
+ empty list `[]`, so each Step's parameter space is initially empty. The scheduler
+ transitions the Job through the following states:
+
+ - `WAITING` — Created, no triggers have fired yet. Tasks: 0.
+ - `RUNNING` — At least one trigger has fired and `Schedule.RunRange` is non-empty.
+ - `COMPLETING` — Either *endAt* has passed, *maxRuns* has been reached, or the Job has
+   been cancelled. No further triggers will be produced. Tasks already triggered may
+   still be running.
+ - Terminal (`SUCCEEDED` / `FAILED` / `CANCELLED`) — Reached when in `COMPLETING` and all
+   triggered Tasks have reached a terminal state.
```

## Design Choice Rationale

### Cron expression as the only schedule type

A 5-field cron expression covers the vast majority of recurring schedules users actually
write — "every N minutes", "every hour", "every day at HH:MM", "every Monday", "the first
of every month at midnight". It is widely understood, parseable by automated tooling, and
human-readable. The `<Schedule>` object is a flat record rather than a polymorphic
`type:`-tagged union so that v1 stays small. A future RFC can introduce alternative
trigger types (interval-only, event-driven, calendar-window) by adding sibling fields and
deprecating *cron* if needed.

### Append-only `Schedule.RunRange` instead of mutable parameter spaces

OpenJD's existing model treats a Step's parameter space as immutable after job creation.
Making it mutable in general would invalidate many invariants — running tasks could see
their indices renumbered, schedulers could not durably persist the parameter space, and
re-evaluation would have to consider arbitrary changes. An append-only list of integers
is the smallest possible weakening of that invariant: existing tasks keep their indices
and parameter values, and the scheduler only needs to compute the new tail of the parameter
space on each trigger.

### Required `Run` task parameter on every step

Requiring every step to bind a task parameter to `Schedule.RunRange` makes the contract
explicit: each step is "per-run", and step dependencies between two scheduled steps
naturally line up by run number. The alternative — letting some steps be "once" and others
"per-run" — adds a coupling between the two that is hard to reason about in general (when
does the once-step run? does it block all runs?) and is left to a follow-up RFC.

### Dynamic values restricted to task parameter ranges

`Schedule.RunRange` is the only template value in the specification whose type is not
fixed at job creation time. Allowing it in arbitrary format strings would force the
scheduler to re-resolve, for example, the Job's *name* on every trigger and decide what
that means. Restricting `Schedule.RunRange` to task-parameter *range* properties — the
exact place where a growing list is meaningful — keeps the rest of the resolution model
unchanged. `Schedule.RunTimestamps` is similarly restricted to Step Script contexts so
the scheduler can ignore them outside that scope.

### `endAt` is required, `startAt` is optional

Every scheduled job is bounded. Requiring *endAt* prevents the common operational mistake
of submitting a job that runs forever and accumulates tasks indefinitely. *startAt* is
optional because "start now" is the overwhelmingly common case; users who need to align
with a future calendar event (a release, a season, a rollout) can still set it.

### `catchup` defaults to `NONE`

A scheduler outage is the most common reason for a missed trigger. The safe default is to
drop missed triggers — replaying them after the fact often does the wrong thing (e.g.,
re-running expired health probes, double-publishing data). Users who explicitly want
backfill can opt in to `LATEST` or `ALL`.

### Built-in `Run` numbers start at 1

Run numbers are user-visible and frequently appear in filenames, log lines, and reports.
Starting at 1 matches how humans count and how most cron-driven systems present run
counts. The implementation can store a 0-indexed list internally; the values exposed to
the template are 1-indexed.

## Prior Art

- **cron / systemd timers / Kubernetes CronJob.** All express schedules as cron expressions
  with optional concurrency and missed-run behaviour. Kubernetes' `CronJob` adds
  `concurrencyPolicy`, `startingDeadlineSeconds`, and `successfulJobsHistoryLimit`. This
  RFC's `catchup` plays the same role as the Kubernetes "starting deadline" but in a more
  user-facing way.
- **Apache Airflow.** Airflow's DAG schedule (`schedule_interval`, `start_date`, `end_date`)
  is the closest prior art to this proposal. Airflow exposes per-run context as
  `{{ ds }}`, `{{ execution_date }}`, etc., and DAG runs accumulate over time as a
  sequence keyed by the trigger time. `Schedule.RunTimestamps[Task.Param.Run - 1]` is
  the OpenJD equivalent of Airflow's `execution_date`.
- **AWS EventBridge Scheduler.** Confirms the cron-or-rate split that we may want to add
  in a follow-up. Demonstrates that timezone support is table stakes for a production
  scheduler — `timezone` is a required field for non-UTC schedules in EventBridge.
- **OpenCue.** OpenCue does not natively support recurring jobs; users wrap submission in
  external schedulers. Conversations with OpenCue users (see GitHub Discussions) indicate
  this is a frequent pain point.
- **Render-management products.** Most commercial render managers either lack scheduling or
  bolt it on as a separate "submit on schedule" feature outside the job template, which
  is the gap this RFC closes.

## Implementation Impact

### Python (openjd-model, openjd-sessions, openjd-cli)

- **openjd-model.** New `Schedule` Pydantic model attached to the Job Template root,
  guarded by the `SCHEDULE` extension. Validation:
  - Cron expression parsing (a small dependency such as `croniter` would suffice).
  - Timezone name validation against the `zoneinfo` database.
  - Cross-validation between *startAt* and *endAt*.
  - Per-step constraint that one task parameter's *range* is `{{Schedule.RunRange}}`.
  - The dynamic `Schedule.RunRange` and `Schedule.RunTimestamps` references must only
    appear in permitted contexts.
  Estimated work: medium. The validator changes are localized.
- **openjd-sessions.** Mostly unaffected. Sessions run a single Task at a time and never
  see the schedule state directly; `Task.Param.Run` is just an integer parameter.
- **openjd-cli.** `openjd run` and `openjd check` need a way to simulate a schedule for
  local testing. A reasonable v1 is `--simulate-runs N` which sets
  `Schedule.RunRange = [1..N]` and `Schedule.RunTimestamps` to fabricated timestamps so
  templates can be exercised end-to-end without a live scheduler. `openjd check` validates
  the new constraints with no runtime simulation.

### Rust (openjd-rs)

- **openjd-model (Rust).** Same shape of change: new struct, schema additions, validators.
  Rust requires picking cron and timezone crates (`cron` + `chrono-tz` are the obvious
  choices). Estimated work: medium.
- **openjd-sessions (Rust).** Unaffected, same as Python.
- **openjd-cli (Rust).** Same `--simulate-runs` flag as the Python CLI for parity.

### Performance implications

The scheduler's trigger handler must, on each cron firing, re-resolve only the task
parameter *range* fields that depend on `Schedule.RunRange`. This is `O(steps)` work per
trigger and produces `O(tasks_per_run)` new tasks, which is well under the cost of a
typical job submission. There is no impact on per-task hot paths.

### Are any features harder in one language than the other?

No. Both ecosystems have mature cron and timezone libraries. The append-only resolution
model is straightforward in both languages.

## Rejected Ideas

### `type:`-tagged polymorphic schedules in v1

Considered making `<Schedule>` a tagged union (`type: CRON`, `type: INTERVAL`, ...) so
that future schedule kinds can slot in without a schema migration. Rejected for v1 to
keep the surface area small. A future RFC can introduce a `type` field with `CRON` as
the default, preserving backward compatibility.

### Per-step `schedule` declarations

Considered allowing `schedule` to be set on individual steps so that a Job could mix
"runs once" steps with "runs on schedule" steps. Rejected because the cross-step
semantics (does a once-step block? run at every trigger? run at the first trigger?)
have no obviously-right answer, and because the most common shape — every step is
per-run and aligned by run number — is well-served by the simpler design. A targeted
follow-up can introduce a step-level `oncePerJob: true` flag if real workloads need it.

### Letting `Schedule.RunRange` be referenced anywhere

Allowing the dynamic list in any format string would mean the scheduler has to
re-resolve, for example, the Job's name, a step's host requirements, or an environment
script on every trigger — and decide what it means to mutate them after tasks are
already running. The narrow restriction to task parameter ranges (and `RunTimestamps`
to step scripts) sidesteps that question entirely.

### Using `RANGE_EXPR` instead of `list[int]` for `Schedule.RunRange`

Both work for the v1 use case (binding to an `INT` task parameter's *range*). A
`list[int]` was chosen because it composes naturally with the EXPR extension —
`len(Schedule.RunRange)`, `Schedule.RunRange[-1]`, and slicing all become available —
and because the parallel `Schedule.RunTimestamps` is naturally a `list[string]`, so
keeping the two consistent is clearer than mixing types.

### Generating run numbers from trigger timestamps directly

Considered using ISO 8601 timestamps as the per-run identifier instead of an integer
counter. Rejected because timestamps are awkward in filenames and shell pipelines, and
because integer run numbers are stable across any schedule edits a future RFC might
allow. Timestamps are exposed alongside via `Schedule.RunTimestamps` for users who need
them.

## Open Questions

### Concurrent runs

If a previous run's tasks have not finished by the time the next trigger fires, what
should happen? Three reasonable behaviours: (a) start the new run anyway, (b) skip the
new trigger, (c) queue the new trigger and start it when the previous run finishes.
This RFC currently says (a) by virtue of standard task-scheduling semantics, but a
`concurrencyPolicy: ALLOW | SKIP | QUEUE` field should be considered before final
comments.

### Pause / resume

Render management systems commonly let an operator pause a job. For a scheduled job,
"pause" almost certainly means "stop firing triggers until I resume", and the *catchup*
mode then determines what happens to triggers that elapsed during the pause. We should
confirm that interpretation before final comments.

### Step-level once-per-job steps

The current design forces every step to be per-run. Real pipelines often have a
"warm-up" step (publish a manifest, prepare a destination) and a "tear-down" step
(notify, archive) that should run once for the whole scheduled job. Should we add a
follow-up RFC for `oncePerJob: true`, or address it in v1?

### Modifying the schedule mid-job

Should `endAt` be extendable after submission? Should `cron` be editable? Both have
legitimate use cases (extending a successful run, adjusting the cadence after seeing
load) but require careful thought about how `Schedule.RunTimestamps` reconciles with
already-fired triggers. Out of scope for v1; called out for awareness.

## Copyright

This document is placed in the public domain or under the CC0-1.0-Universal license, whichever is more permissive.
