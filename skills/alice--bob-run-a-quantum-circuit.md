---
generated: '2026-08-06'
name: Run a quantum circuit on Felis Cloud
method: generated
description: Create a Felis Cloud job against an Alice & Bob emulator or Boson 4 QPU, upload the circuit as QIR, poll until it terminates, and download the results.
api: openapi/alice--bob-felis-cloud-openapi.json
operations: [create_job_v1_jobs__post, upload_input_v1_jobs__job_id__input_post, get_job_v1_jobs__job_id__get, download_output_v1_jobs__job_id__output_get, download_memory_v1_jobs__job_id__memory_get, get_job_metrics_v1_jobs__job_id__metrics_get, cancel_job_v1_jobs__job_id__delete]
source: >-
  Grounded in openapi/alice--bob-felis-cloud-openapi.json — every operationId above was
  verified verbatim in the spec. Auth per authentication/alice--bob-authentication.yml,
  runtime rules per conventions/alice--bob-conventions.yml, failure handling per
  errors/alice--bob-problem-types.yml.
---

# Run a quantum circuit on Felis Cloud

The end-to-end execution flow. Job creation and circuit upload are two separate requests — the
job exists first, and uploading the input is what triggers execution.

## Auth
- Send `Authorization: Basic <API key>` with the **raw** key string. It is not base64-encoded
  RFC 7617 credentials, despite the `Basic` token. See `authentication/alice--bob-authentication.yml`.
- Base URL: `https://api-gcp.alice-bob.com/v1`.
- Every path on this host returns `401 {"error":{"code":401,"message":"Unauthorized"}}` without
  a valid key.

## Before you start
Pick and validate the target first — see `alice--bob-select-and-check-a-backend.md`. Submitting
to an unknown target name returns `422`.

## Steps
1. **Create the job** — `create_job_v1_jobs__post` (`POST /v1/jobs/`) with a `CreateExternalJob`
   body: `inputDataFormat` (`HUMAN_QIR`), `outputDataFormat` (`HISTOGRAM`), `target` (e.g.
   `EMU:6Q:PHYSICAL_CATS`), and `inputParams` (the per-target tunables — read the allowed keys,
   types, defaults and range constraints from `list_targets_v1_targets__get`). The schema sets
   `additionalProperties: false`, so any unknown field is rejected with `422`. Capture the
   returned `id` (a UUID).
2. **Upload the circuit** — `upload_input_v1_jobs__job_id__input_post`
   (`POST /v1/jobs/{job_id}/input`, `multipart/form-data`) with the circuit in QIR format. This
   is the call that starts execution, provided the target is available.
3. **Poll the job** — `get_job_v1_jobs__job_id__get` (`GET /v1/jobs/{job_id}`). Read
   `events[]`; the last `JobEvent.type` is the authoritative state. Keep polling while the state
   is one of `CREATED`, `FETCHING_INPUT`, `INPUT_READY`, `COMPILING`, `COMPILED`, `TRANSPILING`,
   `TRANSPILED`, `EXECUTING`.
4. **On `SUCCEEDED`, read the results** — `download_output_v1_jobs__job_id__output_get`
   (`GET /v1/jobs/{job_id}/output`) returns the measurement histogram (bitstrings to
   occurrence counts). If you need per-shot data instead, use
   `download_memory_v1_jobs__job_id__memory_get` (`GET /v1/jobs/{job_id}/memory`).
5. **Record the cost basis** — `get_job_metrics_v1_jobs__job_id__metrics_get`
   (`GET /v1/jobs/{job_id}/metrics`) returns `qpu_duration_ns` and `simulation_duration_ns`.
   Felis Cloud bills by wall-clock backend time, so this is the number to log.
6. **Abandon cleanly if needed** — `cancel_job_v1_jobs__job_id__delete`
   (`DELETE /v1/jobs/{job_id}`) cancels an active job. The job's last event becomes `CANCELLED`.

## Cost and retry safety — read this before automating
- The API has **no idempotency mechanism**. A retried `POST /v1/jobs/` creates a second job and
  a second billable execution. De-duplicate on your side by keying on your own request id and
  storing the returned job `id` before you retry anything.
- Boson 4 QPU time is $5,000/hour beyond the 1 free hour per month; emulators are $25/hour
  beyond theirs. Develop against `EMU:*` targets, or against the free in-process
  `AliceBobLocalProvider`, before ever selecting a `QPU:*` target. See
  `sandbox/alice--bob-sandbox.yml` and `plans/alice--bob-plans.yml`.
- Jobs are capped at **15 minutes**. Split long experiments.

## Failure handling
Failures after acceptance arrive **in-band on a 200**, in `events[]` / `errors[]`, not as HTTP
error statuses. Treat these as terminal:
- `COMPILATION_FAILED` — the QIR did not compile. Re-check it against the QIR spec.
- `TRANSPILATION_FAILED` — the circuit uses a gate the target does not implement. Physical
  cat-qubit backends deliberately omit Hadamard because it is not bias-preserving; use
  `initialize('+', q)` and `measure_x(q, c)` instead. Check
  `TargetConfiguration.instructions` from `list_targets_v1_targets__get`.
- `EXECUTION_FAILED`, `TIMED_OUT`, `CANCELLED` — see `errors/alice--bob-problem-types.yml`.

Request-level errors are the FastAPI envelope `{"detail": [{"loc": [...], "msg": "...",
"type": "..."}]}` with status `422`. Note that no `404` is declared for an unknown `job_id` —
handle an unexpected status defensively.

## Reference implementation
`qiskit-alice-bob-provider` (`pip install qiskit-alice-bob-provider`) wraps all of the above;
Alice & Bob's own docs call it the reference implementation of this API.
