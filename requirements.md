# TaskForge — MVP Requirements

## 1. Purpose

Build a small asynchronous job-processing system using Java 21, Spring Boot, PostgreSQL, and Kafka. The MVP must demonstrate:

```text
Submit a job → persist → dispatch → execute → retrieve its result
```

## 2. Architecture

- A Spring Boot API accepts jobs and returns their IDs.
- PostgreSQL stores jobs, execution results, and outgoing dispatch messages in an outbox table.
- An outbox publisher runs within the API application and sends pending messages to Kafka.
- One worker consumes messages from one job topic and executes one simulated handler.
- Clients poll the API for status and results.

```text
Client → API → PostgreSQL (job + outbox in one transaction)
                  Outbox publisher → Kafka → Worker
                                               ↓
                                           PostgreSQL
Client → API → job status and result
```

## 3. Job Model and Execution

Store a unique job ID, type, payload, status, creation/update timestamps, start/completion timestamps, result, and error information when applicable.

The only supported states and transitions are:

```text
QUEUED → RUNNING → SUCCEEDED
                → FAILED
```

- Implement one simulated handler, `demo.echo`, which returns the submitted payload as its result without external side effects.
- The worker must atomically claim a `QUEUED` job before executing it and persist the terminal status with its result or error.
- A handler exception marks the job `FAILED`; failed jobs are not automatically retried.
- State changes must be validated. Completed jobs must not be executed again when a duplicate message arrives.

## 4. API

The only job endpoints are:

### Submit a job

```http
POST /api/v1/jobs
```

Example request:

```json
{
  "type": "demo.echo",
  "payload": { "message": "Hello" }
}
```

- Validate JSON, the supported job type, and the handler's payload requirements.
- Save the job as `QUEUED` and its outbox message in one database transaction.
- Return `202 Accepted` with the job ID only after the transaction commits; do not wait for execution.
- Invalid requests return `400 Bad Request` with a clear error response.

### Retrieve status and result

```http
GET /api/v1/jobs/{id}
```

- Return the job ID, type, status, timestamps, result when successful, and error when failed.
- Return `404 Not Found` for an unknown job ID.
- Clients poll this endpoint; the MVP does not push results back to clients.

## 5. Transactional Outbox and Duplicate Safety

The outgoing message tells the worker that a job is ready to process and contains its job ID.

1. Save the job and outbox record together: either both commit or neither commits.
2. The publisher reads pending outbox records and sends their messages to Kafka.
3. Mark a record published only after Kafka acknowledges the send.
4. If publication fails, leave the record pending for a later publishing attempt.

Kafka delivery may repeat, including if the publisher crashes after sending but before recording publication. The worker must safely ignore duplicate messages for jobs already claimed or completed and acknowledge processed messages only after the outcome is persisted. Republishing an outbox message is distinct from retrying a failed job.

## 6. Local Operation

- Run the API with its outbox publisher, one worker, PostgreSQL, and Kafka through `docker compose up`.
- Emit basic logs for submission, dispatch, execution, success, and failure, including the job ID where applicable.
- Provide basic health checks for the API and worker, with dependency availability visible.

## 7. Verification and Definition of Done

Use an end-to-end integration test with real PostgreSQL and Kafka instances to verify that a submitted job is persisted, dispatched through the outbox, executed, and returned as `SUCCEEDED` with its result through the status endpoint.

Also verify:

- Invalid input is rejected and unknown job IDs return `404`.
- A handler exception produces `FAILED` with error information.
- Duplicate delivery does not execute a completed job again or change its result.
- If Kafka is temporarily unavailable, the outbox record remains pending and is published after Kafka recovers.

The MVP is complete when the full flow works locally through Docker Compose, these checks pass, and the limitations below are documented in the usage instructions.

## 8. MVP Limitations

- No automatic job retries, dead-letter queue, leases, or execution timeouts. A worker crash or hung handler may leave a job `RUNNING`; automatic recovery is outside the MVP.
- Duplicate-safe message processing does not promise exactly-once execution or deduplicate repeated client submissions. Separate submissions create separate jobs.
- No cancellation, job listing, multiple workers, or additional handlers.
- No dashboards, distributed tracing, load or throughput targets, cloud deployment, or Terraform.

The MVP demonstrates durable acceptance and asynchronous dispatch; it does not claim production-grade recovery guarantees.
