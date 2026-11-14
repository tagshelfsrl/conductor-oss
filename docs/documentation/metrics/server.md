---
description: "Server Metrics — monitor Conductor server health and performance using Micrometer-based metrics and alerting."
---
# Server Metrics

!!! Info "Feature Update"
    Since [v3.21.16](https://github.com/conductor-oss/conductor/releases/tag/v3.21.16), Conductor has switched to [Micrometer](https://micrometer.io/) for metrics collection.


Conductor uses [Micrometer](https://micrometer.io/) for metrics collection and export. 

The following metrics are published by the Conductor server. You can export these metrics to set up alerts for your workflows and tasks.

| Metric Name          | Description       | Tags  |
| ------------- |:----------------- | ----- |
| workflow_server_error | The rate at which server-side errors are occurring.  | methodName|
| workflow_failure | The number of failed workflows.                           |workflowName, status|
| workflow_start_error | The number of workflows that fail to start.           |workflowName|
| workflow_running | The number of running workflows.                          | workflowName, version|
| workflow_execution | The time taken for workflow completion.                 | workflowName, ownerApp |
| task_queue_wait | The amount of time spent by a task in queue.               | taskType |
| system_task_first_evaluation_delay | Scheduled timestamp to first system-task executor evaluation, milliseconds at recording | taskType |
| system_task_phase | Monotonic time spent loading, evaluating, polling or retiring an absent system task; timer count also records outcomes | taskType, phase, outcome |
| system_task_available_slots | Distribution of available executor slots at each system-task poll | taskType |
| system_task_poll_batch | Distribution of task references returned by a system-task poll | taskType |
| task_execution | The time taken to execute a task.                           | taskType, includeRetries, status |
| task_poll | The time taken to poll for a task.                               | taskType|
| task_poll_count | The number of times the task is being polled.              | taskType, domain |
| task_queue_depth | The queue depth for pending tasks.                        | taskType, ownerApp |
| task_rate_limited | The current number of tasks that are being rate limited. | taskType |
| task_concurrent_execution_limited | The current number of tasks that are being limited by its concurrent execution limit. | taskType |
| task_timeout | The number of timed-out tasks. | taskType |
| task_response_timeout | The number of tasks that timed out due to `responseTimeout`. | taskType |
| task_update_conflict | The number of task update conflicts. <br/><br/> For example, a worker updates the task status even though the workflow is already in a terminal state. | workflowName, taskType, taskStatus, workflowStatus |
| event_queue_messages_processed | The number of messages fetched from an event queue. | queueType, queueName |
| observable_queue_error | The number of errors encountered when fetching messages from an event queue. | queueType |
| event_queue_messages_handled | The number of messages executed from an event queue. | queueType, queueName |
| external_payload_storage_usage | The number of times an external payload storage was used. | name, operation, payloadType |


## Supported monitoring systems

Conductor supports the following Micrometer publishers:

- [Atlas](https://docs.micrometer.io/micrometer/reference/implementations/atlas.html)
- [Prometheus](https://docs.micrometer.io/micrometer/reference/implementations/prometheus.html)
- [Datadog](https://docs.micrometer.io/micrometer/reference/implementations/datadog.html)
- [JMX](https://docs.micrometer.io/micrometer/reference/implementations/jmx.html)
- [OpenTelemetry Protocol (OTPL)](https://docs.micrometer.io/micrometer/reference/implementations/otlp.html)
- [Dynatrace](https://docs.micrometer.io/micrometer/reference/implementations/dynatrace.html)
- [Elasticsearch](https://docs.micrometer.io/micrometer/reference/implementations/elastic.html)
- [New Relic](https://docs.micrometer.io/micrometer/reference/implementations/new-relic.html)
- [StackDriver](https://docs.micrometer.io/micrometer/reference/implementations/stackdriver.html)
- [StatsD](https://docs.micrometer.io/micrometer/reference/implementations/statsD.html)
- [CloudWatch](https://docs.micrometer.io/micrometer/reference/implementations/cloudwatch.html)
- [Azure Monitor](https://docs.micrometer.io/micrometer/reference/implementations/azure-monitor.html)
- [Influx](https://docs.micrometer.io/micrometer/reference/implementations/influx.html)

### Enabling metrics collection

To enable metrics collection to a particular monitoring system, refer to the [Micrometer documentation](https://docs.micrometer.io/micrometer/reference/implementations.html) complete the implementation. You will also need to enable the particular monitoring system in the Conductor's [`application.properties` file](https://github.com/conductor-oss/conductor/blob/6147d61d1babf47f5a0a328d114f1eb5d3d5ecb1/server/src/main/resources/application.properties#L163).

## System-task scheduling and absent tasks

The system-task worker supplies the exact polled queue to its executor, including
namespace and isolation suffixes. When the authoritative task lookup returns
null, the executor removes that reference from that queue. A lookup that throws
leaves the reference intact for retry; it is never treated as proof that a task
is absent. Failed removals remain retryable. This applies only to individually
polled references and performs no bulk scan, purge or workflow deletion.

The two-argument executor entry point remains compatible, but cannot retire an
absent task without its queue identity. Loaded tasks keep their existing
execution, callback, persistence and workflow-decision behavior.

`system_task_phase` has static phases `load`, `orphan_remove`, `evaluate` and
`poll`. Lookup outcomes distinguish `found`, `missing` and `failed`; removal
outcomes distinguish `completed` and `failed`. Evaluation includes the existing
workflow lookup, system-task execution, queue update, persistence and decision,
and is labelled with the resulting task status. Poll outcomes are `empty`,
`acquired`, `limited` or `failed`. Timers use nanoseconds internally and the
existing Spectator exporter controls display units. No task or workflow ids are
metric labels, and metrics failures do not alter processing.

`JOIN.startTime` is set by its mapper and is not an executor pickup timestamp.
Use `system_task_first_evaluation_delay` to observe initial scheduling latency
when the existing poll counter first advances after admission checks,
and compare JOIN completion with the completion of its joined branches to
measure residual wait. First-evaluation delay uses server wall-clock timestamps;
phase durations use a local monotonic clock. Negative wall-clock differences
are clamped to zero for the timer. Callback duration is a recheck
delay, not an upper bound on queue residence or total JOIN latency.
Concurrency/rate-limit deferrals do not emit repeated first-evaluation samples.
A single `System task first evaluation` information log also carries task and
workflow ids, scheduled and observed epoch-millisecond UTC timestamps, and the
raw delay (including a negative value if clocks disagree). This supports exact
workflow correlation through the existing logs independently of metric dashboards.

The new metrics use existing collection and export paths without extra queue or
datastore requests. Deploy and benchmark in staging before promotion. This
change does not directly repair any existing environment; confirmed absent
references are retired naturally only when the updated worker polls them.
