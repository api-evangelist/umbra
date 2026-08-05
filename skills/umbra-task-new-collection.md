---
name: Task a new SAR collection with Umbra Canopy
description: >-
  Check whether Umbra's SAR constellation can image a target in a time window, then submit a Task
  against a real opportunity and follow it through to delivered imagery. This is the marquee Canopy
  flow and the only one that spends money.
api: openapi/umbra-tasking-openapi.yml
operations:
  - create_feasibility
  - get_feasibility
  - get_restricted_access_areas
  - create_task
  - get_task
  - get_collect
  - cancel_task
generated: '2026-08-05'
method: generated
source: openapi/umbra-tasking-openapi.yml + https://docs.canopy.umbra.space/docs/task-lifecycle-tutorial
---

# Task a new SAR collection with Umbra Canopy

Base URL `https://api.canopy.umbra.space/tasking`.
Sandbox `https://api.canopy.prod.umbra-sandbox.space/tasking` — **develop here first**, it is
enabled by default and produces no billable Collects.

## Before you start

Get a bearer token. Either copy a 24-hour token from `https://canopy.umbra.space/account`, or run
the client-credentials exchange:

```
POST https://auth.canopy.umbra.space/oauth/token
Content-Type: application/json
{"client_id":"…","client_secret":"…","audience":"https://api.canopy.umbra.space","grant_type":"client_credentials"}
```

Send it as `Authorization: Bearer <access_token>` on every request. **Cache it.** The exchange is
capped at 50 per rolling 24 hours per client, and going over returns HTTP **400** with the real
code buried at `error_description.code` = 429.

For the sandbox, use audience `https://api.canopy.prod.umbra-sandbox.space` instead.

## Step 1 — confirm the target is taskable

Call `get_restricted_access_areas`. It returns a GeoJSON FeatureCollection of locations where
tasking is disallowed **for your organization**. If your target intersects one, stop here — a Task
there will be rejected.

## Step 2 — ask for a feasibility

Call `create_feasibility` with `imagingMode` (`SPOTLIGHT` or `SCAN`), `windowStartAt`,
`windowEndAt`, and the matching constraints object (`spotlightConstraints` or `scanConstraints`)
carrying a GeoJSON `geometry`.

It returns a Feasibility with `status: RECEIVED`. **It is asynchronous.**

## Step 3 — poll for opportunities

Poll `get_feasibility` by id until `status` is `COMPLETED`, at which point the `opportunities`
array is populated. `ERROR` is terminal.

Each Opportunity carries `windowStartAt`, `windowEndAt`, `durationSec`, `satelliteId` and the
radar geometry (grazing angle, target azimuth, squint, slant range). Pick one that meets your
imaging requirements — grazing angle and squint drive image quality.

Rate limit: `create_feasibility` is **2/second, 120/minute, 3600/hour**. Do not fan out.

## Step 4 — submit the Task

Call `create_task` with a window that falls inside your chosen Opportunity:

- `imagingMode` — `SPOTLIGHT` or `SCAN`
- `windowStartAt` / `windowEndAt`
- `spotlightConstraints` or `scanConstraints` with the GeoJSON `geometry`
- `productTypes` — from `CPHD`, `GEC`, `SIDD`, `SICD`, `DI_GIF`, `DI_TIF`, `CRSD`
- `deliveryConfigId` — optional; without it, data is downloaded from Canopy instead of pushed
- `taskName`, `tags`, `userOrderId` — your own labels
- `userOrderId` **matters**: see the retry warning below

The Task comes back `ACTIVE` and moves to `SUBMITTED` or `REJECTED` within a few minutes.

> **There is no idempotency key.** `create_task` is a non-idempotent POST and a Collect is
> billable. If the call times out, do **not** blind-retry. Set a unique `userOrderId` on every
> submission, then call `post_search_tasks` filtering on `userOrderId` (`eq`) to check whether the
> Task already landed before submitting again.

Rate limit: `create_task` is **2/second, 120/minute, 300/hour**.

## Step 5 — follow the Task

Poll `get_task` by id. Status walks `RECEIVED → REVIEW → ACCEPTED → ACTIVE → SCHEDULED → TASKED →
PROCESSING → PROCESSED → DELIVERING → DELIVERED`. Terminal failures are `REJECTED`, `EXPIRED`,
`CANCELED`, `ERROR`, `ANOMALY`, `INCOMPLETE`.

Read `properties.collectIds` once populated — that is the fan-out into Collects.

**Back off, do not poll harder.** Under load, transitions can take more than a few minutes; Umbra
asks you to reduce polling rate. Delays beyond an hour warrant contacting a Canopy representative.

## Step 6 — read the Collects

Call `get_collect` for each id in `properties.collectIds`. Collect status walks `SCHEDULED →
COMMANDED → COLLECTED → DOWNLINKED → PROCESSING → PROCESSED → DELIVERING → DELIVERED`.

Once `PROCESSED`, `archiveIds` is your join into the catalog — use those ids against the STAC APIs
to fetch the imagery assets (see `umbra-search-archive-imagery.md`).

## Cancelling

`cancel_task` (PATCH `/tasks/{id}/cancel`) works only while status is one of `SCHEDULED`,
`SUBMITTED`, `ACCEPTED`, `ACTIVE`. Otherwise it returns **422**. Same 2/second limit.

## Error handling

- **422** — `HTTPValidationError`. Read `detail[].loc` for the offending field and `detail[].type`
  for the reason. On `cancel_task` it also means the Task is not in a cancellable state.
- **401** — token missing, expired, or minted for the wrong audience (live token against sandbox
  host, or vice versa). Re-mint and retry. Undeclared in the spec but real.
- **429** — back off with exponential delay plus jitter. For the 2/second operations, queue
  client-side and skip requests whose Task window has already passed.

## Testing this flow safely

In the sandbox, statuses transition in seconds rather than days, and Feasibility opportunities are
static (they do not reflect the constraints you sent). To exercise the QA-failure path, put the
string `umbra:delivery:qa:failure` anywhere in `taskName`.
