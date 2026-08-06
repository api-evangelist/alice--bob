---
generated: '2026-08-06'
name: Select and check a Felis Cloud backend
method: generated
description: Discover Alice & Bob's emulator and QPU targets, read the instructions and tunable parameters each supports, and confirm it is up and scheduled before spending money on a job.
api: openapi/alice--bob-felis-cloud-openapi.json
operations: [list_targets_v1_targets__get, target_status_v1_targets__target__health_get, list_target_availabilities_v1_targets__target__availabilities_get, check_health_v1_health__get, list_jobs_v1_jobs__get]
source: >-
  Grounded in openapi/alice--bob-felis-cloud-openapi.json — every operationId above was
  verified verbatim in the spec. Target names and the naming convention are from
  https://felis.alice-bob.com/docs/backends/about_backends/; the entity graph is
  data-model/alice--bob-data-model.yml.
---

# Select and check a Felis Cloud backend

Target selection is the decision that determines whether a job costs $25/hour or $5,000/hour,
and whether it can run at all. Do this before `create_job`.

## Auth
- `Authorization: Basic <API key>` (raw key, not base64). Base URL `https://api-gcp.alice-bob.com/v1`.
- Note that `list_targets_v1_targets__get` is the one operation in the spec with no
  `authorization` parameter declared — but the host still requires the key, like every other
  path. See `authentication/alice--bob-authentication.yml`.

## Steps
1. **Check the service is up** — `check_health_v1_health__get` (`GET /v1/health/`) returns
   `"OK"` when the Felis Cloud API is running.
2. **List the backends** — `list_targets_v1_targets__get` (`GET /v1/targets/`). Each
   `TargetConfiguration` carries:
   - `name` — the value you pass as `target` on job creation, formatted `[EMU|QPU]:<n>Q:<NAME>`
     (`EMU` = emulator, `QPU` = real chip, `<n>Q` = maximum circuit width).
   - `numQubits` — the ceiling on circuit width.
   - `instructions[]` — the gate signatures this backend implements. **Check your circuit's
     gates against this list.** Physical cat-qubit backends have no Hadamard by design.
   - `inputParams` — a map of `InputParamConfiguration`, each with `type` (`float`/`int`/`bool`),
     `required`, `default` and `constraints[]` range bounds. These are the values you put in the
     job's `inputParams` object (e.g. `average_nb_photons`, `kappa_1`, `kappa_2`).
3. **Check the target's health** — `target_status_v1_targets__target__health_get`
   (`GET /v1/targets/{target}/health`) returns a `TargetStatus`: `OK` (enabled and working),
   `NOK` (enabled but not working) or `OFF` (disabled). Do not submit against `NOK` or `OFF`.
4. **Check the schedule** — `list_target_availabilities_v1_targets__target__availabilities_get`
   (`GET /v1/targets/{target}/availabilities`, paginated with `limit`/`offset`). Each
   `TargetAvailability` carries `enabled`, an optional operator `message`, `timestamp` and
   `createdAt`. Hardware runs on a published schedule and Alice & Bob states it may take a chip
   down without notice, so this is the operational truth for a `QPU:*` target.
5. **See what you already have queued** — `list_jobs_v1_jobs__get` (`GET /v1/jobs/`, paginated
   with `page`/`limit`, limit max 1000) returns your active and completed jobs, so you do not
   re-submit work that is already running.

## Choosing the right class of target
- **Free, no account** — the in-process `AliceBobLocalProvider` emulators. Use these while
  writing the circuit.
- **`EMU:*` on the API** — same contract, same auth, same job lifecycle as hardware, at
  $25/hour beyond the free monthly hour. Use these to validate an integration end-to-end.
- **`QPU:*` on the API** — real Boson 4 chips at $5,000/hour beyond the free monthly hour.
  Only after both of the above pass.

See `sandbox/alice--bob-sandbox.yml` for the full target list and `plans/alice--bob-plans.yml`
for the rates.

## Notes
- `list_targets_v1_targets__get` declares only a `200` — no `422` — so an unfiltered list is the
  safe way to enumerate. `target_status_...` and `list_target_availabilities_...` return `422`
  for an unknown target name.
- Neither list operation returns a total count, a `has_more` flag or a next link; you have
  reached the end when you get back fewer items than `limit`. See
  `conventions/alice--bob-conventions.yml`.
- There is no webhook or event stream for a target going offline — this is a polling surface.
  The human-readable equivalents are
  https://felis.alice-bob.com/docs/felis_cloud/hardware_availability_schedule/ and the live
  console status at https://api-gcp.alice-bob.com/console/status.
