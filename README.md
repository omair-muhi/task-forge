# TaskForge

TaskForge is a planned asynchronous job-processing service. Clients submit jobs, receive a job ID, and poll for status and results while a worker executes the jobs in the background.

The repository currently contains the [MVP requirements](requirements.md); application code and deployment configuration have not yet been added.

## Technology Stack

| Role | Technology | Purpose |
| --- | --- | --- |
| Language / runtime | Java 21 | Runs the API and worker applications. |
| Application framework | Spring Boot | Provides the API and application infrastructure. |
| Database | PostgreSQL | Stores jobs, results, and outgoing messages in an outbox table. |
| Messaging | Kafka | Dispatches jobs asynchronously to the worker. |
| Local deployment | Docker Compose | Runs the API, worker, PostgreSQL, and Kafka together in containers for development and testing. |

Docker Compose is the planned local setup. On-prem deployment would run the service on organization-managed servers; cloud deployment would run it on a provider's infrastructure. Both are outside the MVP scope. Versions other than Java 21 are not yet specified.

## Planned Architecture

```text
Client → API → PostgreSQL (job + outbox in one transaction)
                  Outbox publisher → Kafka → Worker
                                               ↓
                                           PostgreSQL
Client → API → job status and result
```

The API saves a job and its dispatch message together. An outbox publisher sends pending messages to Kafka, and the worker atomically claims a queued job before executing it. Duplicate messages must not re-execute jobs that have already been claimed or completed.

## Planned API

| Endpoint | Behavior |
| --- | --- |
| `POST /api/v1/jobs` | Accepts a valid job and returns `202 Accepted` with its ID after persistence. |
| `GET /api/v1/jobs/{id}` | Returns job status and any result or error; returns `404` for an unknown ID. |

The MVP supports one handler, `demo.echo`, which returns the submitted payload as its result. Example submission:

```json
{
  "type": "demo.echo",
  "payload": { "message": "Hello" }
}
```

Jobs move from `QUEUED` to `RUNNING`, then to `SUCCEEDED` or `FAILED`.

## Local Development and Verification

Once application code and Compose configuration are available, the intended startup command is `docker compose up`. It is not runnable from this repository yet.

The planned integration tests use real PostgreSQL and Kafka instances to verify submission through execution and result retrieval, invalid input, unknown job IDs, handler failures, duplicate delivery, and publication after Kafka recovers from an outage.

## MVP Limitations

- No automatic job retries, dead-letter queue, execution timeouts, or recovery of jobs left `RUNNING` by a crashed worker.
- No exactly-once execution guarantee or deduplication of repeated client submissions.
- One worker and one handler; no cancellation or job listing.
- No dashboards, distributed tracing, performance targets, cloud deployment, or Terraform.

See [requirements.md](requirements.md) for the full specification and acceptance criteria.
