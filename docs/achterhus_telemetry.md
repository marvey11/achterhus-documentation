# Achterhus Telemetry

## Purpose and components

The Telemetry application records the lifecycle and health of services running on the
Achterhus home server. It consists of:

- the **Service Orchestrator**, which resolves dependencies, schedules service runs,
  creates containers and supplies each container with its run identity;
- the **Telemetry API**, a FastAPI service that validates and stores run states and
  events in PostgreSQL;
- the **Telemetry Client**, a Python context manager used by services to report
  application status, metrics, logs summaries and events;
- the **Watchguard**, which observes container execution and can report failures such
  as timeouts and out-of-memory termination; and
- the **Telemetry Dashboard**, which reads run history and service summaries from the
  API.

The API and PostgreSQL container communicate over the shared `achterhus-network`.
Containers should use the PostgreSQL service DNS name on that network rather than a
host-only address.

### Service orchestration and dependency resolution

The Service Orchestrator reads a YAML service manifest, validates the DAG and resolves
service execution order using `graphlib.TopologicalSorter`. Each service definition
includes the image reference, optional command, environment variables, Docker volumes,
and the upstream services it must wait for before it can start. The orchestrator only
pulls a new image when the remote GHCR manifest digest differs from the locally cached
image digest, which keeps one-shot service runs efficient while still ensuring the job
uses the latest published image.

## A service run from scheduling to completion

1. The orchestrator creates a UUID for the run. This UUID is the run's stable identity
   across orchestration, application reports, events and dashboard views.
2. Before execution, the orchestrator registers the run with `POST /api/v1/runs`.
   The API records the service name, UUID, initial `SCHEDULED` state and a
   `status_changed` event.
3. As execution progresses, the orchestrator patches the run to `IMAGE_PULLING` and
   `STARTING` when those phases apply. A service container receives the API URL,
   service name and `SERVICE_RUN_ID` through its environment.
4. The Python service creates a `TelemetryClient`. The context manager reports
   `INITIALIZING` followed by `RUNNING` on entry, then `SUCCESS` on normal exit or
   `FAILED` if an exception escapes. Services can report intermediate status updates
   and run-scoped events.
5. The Watchguard reports outcomes it observes, including `TIMEOUT` and `OOM_KILLED`.
   The orchestrator reports container-creation and host/daemon errors.
6. The API stores each accepted status change in the run record and in its event
   history. The dashboard reads the current run state, event history and service
   overview from the API.

### Run identity and container environment

The orchestrator must use the same UUID when registering a run and when starting its
container. The container's `SERVICE_RUN_ID` must be that registered UUID. A service
must not generate a replacement run ID or register a second run. The client accepts
`SERVICE_RUN_ID` automatically; an explicit `run_id` constructor argument is
available for controlled integrations and tests.

The service also needs its `service_name` and a reachable API URL when constructing
the client. These are deployment configuration, not secrets. PostgreSQL credentials
are secrets and must be injected at runtime.

## Lifecycle state machine

| State | Responsible component | Terminal | Meaning |
| --- | --- | --- | --- |
| `SCHEDULED` | Orchestrator | No | Registered and waiting for its execution turn. |
| `IMAGE_PULLING` | Orchestrator | No | Required image layers are being fetched. |
| `STARTING` | Orchestrator | No | Container creation succeeded and start was issued. |
| `INITIALIZING` | Application | No | Process is loading configuration and preparing its workload. |
| `RUNNING` | Application | No | Core workload logic is active. |
| `SUCCESS` | Application | Yes | Work completed and the process exited successfully. |
| `FAILED` | Application or Watchguard | Yes | An unhandled exception or non-zero exit occurred. |
| `CREATE_FAILED` | Orchestrator | Yes | Container could not be created, for example because an image or mount was invalid. |
| `START_FAILED` | Orchestrator | Yes | Container failed immediately after start was issued. |
| `OOM_KILLED` | Watchguard | Yes | The operating system killed the process after memory exhaustion. |
| `TIMEOUT` | Watchguard or API stale-run check | Yes | The execution limit was exceeded or an older running instance was superseded. |
| `ORCHESTRATOR_ERROR` | Orchestrator | Yes | A host, daemon or orchestrator failure prevented execution. |

Allowed transitions:

| Current state | Next state(s) |
| --- | --- |
| `SCHEDULED` | `IMAGE_PULLING`, `STARTING`, `CREATE_FAILED`, `ORCHESTRATOR_ERROR` |
| `IMAGE_PULLING` | `STARTING`, `CREATE_FAILED`, `ORCHESTRATOR_ERROR` |
| `STARTING` | `INITIALIZING`, `RUNNING`, `START_FAILED`, `OOM_KILLED`, `ORCHESTRATOR_ERROR` |
| `INITIALIZING` | `RUNNING`, `SUCCESS`, `FAILED`, `OOM_KILLED`, `TIMEOUT` |
| `RUNNING` | `SUCCESS`, `FAILED`, `OOM_KILLED`, `TIMEOUT` |
| Any terminal state | No further transition |

The API rejects transitions from terminal states and invalid phase changes with
`422 Unprocessable Entity`. A repeated non-terminal status is accepted as an
idempotent heartbeat and refreshes its timestamp and supplied metadata. The
orchestrator/watchguard can report `FAILED`, `TIMEOUT` or `OOM_KILLED` while the run
is `RUNNING`. When a new run of a service enters `RUNNING`, any previous run of that
service still in `RUNNING` is automatically terminalised as `TIMEOUT`. The API records
this generated status change with source `api` and reason `StaleRunSuperseded`.

The source values are `orchestrator`, `application`, `watchguard` and `api`. Timestamps
are ISO 8601; send UTC timestamps with a `Z` suffix where possible. The API normalises
naive input timestamps to UTC.

## API contract

The API base path is `/api/v1`.

### Register a run

The orchestrator calls `POST /runs` before execution:

```json
{
  "service_name": "newsletter-worker",
  "run_id": "8d4b64d8-7e28-42f8-8cb8-3b4d202d3d83",
  "status": "SCHEDULED",
  "source": "orchestrator",
  "timestamp": "2026-09-29T10:15:00Z"
}
```

`status`, `source` and `timestamp` have API defaults (`SCHEDULED`, `orchestrator`
and the current UTC time). Run IDs are unique; duplicate registration returns `409
Conflict`.

### Report a status

Use `PATCH /runs/{run_id}/status` for lifecycle status and optional data:

```json
{
  "status": "CREATE_FAILED",
  "source": "orchestrator",
  "timestamp": "2026-09-29T10:15:00Z",
  "error_details": {
    "reason": "DockerImageNotFound",
    "message": "The service image could not be pulled"
  }
}
```

`metrics` and `logs_summary` are optional status payload fields. Structured
`error_details` are stored with the current run, and a string `message` is also
exposed as `error_message` for convenient display.

### Record and retrieve events

`POST /runs/{run_id}/events` accepts an `event_type`, `source`, `timestamp` and
JSON-object `details`. For example:

```json
{
  "event_type": "backup_checkpoint",
  "source": "application",
  "timestamp": "2026-09-29T10:15:00Z",
  "details": {
    "files_copied": 120
  }
}
```

`GET /runs/{run_id}/events` returns events in timestamp order. The API also writes a
`status_changed` event for initial registration, every accepted status patch and each
stale-run timeout it generates.

### Read runs and overview

`GET /runs` returns a JSON array of the newest runs. It supports the optional
`service_name` and `status` filters, `limit` (default `50`, maximum `200`) and the
one-based `page` number (default `1`). The response remains an array; pagination
metadata is not currently included.

`GET /overview` returns one summary per service with `last_status`, `last_run_at`,
`total_runs` and `failed_runs`. The failed count includes `FAILED`, `CREATE_FAILED`,
`START_FAILED`, `OOM_KILLED`, `TIMEOUT` and `ORCHESTRATOR_ERROR`.

## PostgreSQL and configuration

PostgreSQL is the only supported runtime database. The API requires `DATABASE_URL`
with an async PostgreSQL driver (normally
`postgresql+asyncpg://<user>:<password>@<host>:5432/<database>`); it has no embedded
database or production fallback. Supply the value via the container environment or
an ignored local `.env` file. Never put passwords in source, checked-in compose files,
screenshots or logs.

The database deployment must create the `telemetry` schema and the dedicated
application role before starting the API. Configure that role's search path to
`telemetry` and grant it the required schema, table and sequence permissions. The
API creates its tables in the active schema on startup; it does not provision the
database, schema or role. PostgreSQL should be healthy before the API starts, and the
API should be restarted if the database becomes unavailable during initialisation.

## Client usage

Inside a Python service container:

```python
from telemetry.client import TelemetryClient

client = TelemetryClient(
    api_url="http://telemetry-api:8000",
    service_name="newsletter-worker",
)

with client:
    client.set_metric("items_processed", 120)
    client.send_event("backup_checkpoint", {"files_copied": 120})
```

The client reads the orchestrator-provided `SERVICE_RUN_ID`, patches status and posts
events. It does not register runs. `start()` reports `INITIALIZING`; entering the
context manager follows it with `RUNNING`, which also activates the stale-run check.
Callers managing lifecycle manually can use `update_status()` when initialisation is
complete. Telemetry transport failures are logged without interrupting the service
workload; services must therefore rely on their own runtime and health checks rather
than treating telemetry delivery as durable.

For API field details and development commands, see the
[Telemetry API README](https://github.com/marvey11/achterhus-telemetry-api/blob/main/README.md) and
[Telemetry Client README](https://github.com/marvey11/achterhus-telemetry-client/blob/main/README.md).
