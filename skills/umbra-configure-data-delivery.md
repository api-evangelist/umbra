---
name: Configure and verify Umbra Canopy data delivery
description: >-
  Set up a DeliveryConfig so Canopy pushes finished SAR products straight into your own AWS S3 or
  Google Cloud Storage bucket, verify it before attaching it to a Task, and manage the
  machine-to-machine credentials the automation runs on.
api: openapi/umbra-delivery-openapi.yml
operations:
  - list_delivery_configs
  - post_delivery_config
  - verify_delivery_config
  - delete_delivery_config
  - get_schema
  - create_token
  - get_token
  - rotate_token
  - delete_token
  - get_organization_settings
  - get_constraints_for_contract_or_org_default
generated: '2026-08-05'
method: generated
source: >-
  openapi/umbra-delivery-openapi.yml + openapi/umbra-admin-openapi.yml +
  https://docs.canopy.umbra.space/docs/delivery-configs
---

# Configure and verify Umbra Canopy data delivery

Two base URLs:

- Delivery — `https://api.canopy.umbra.space/delivery`
- Admin — `https://api.canopy.umbra.space/admin`

Without a DeliveryConfig, finished products are made available for download inside the Canopy app.
With one, Canopy copies them directly into your bucket. This is a **push of data**, not an event
notification — Canopy has no webhooks.

## Part 1 — machine-to-machine credentials

Automation needs client credentials rather than a hand-copied 24-hour token.

- `create_token` (POST `/m2m/token`) — creates the M2M application for your organization
- `get_token` (GET `/m2m/token`) — retrieves the `client_id` and `client_secret`
- `rotate_token` (PUT `/m2m/token`) — rotates them
- `delete_token` (DELETE `/m2m/token`) — deletes the application

> **These are organization-wide, and there is exactly one set.** Any user in the org can rotate or
> delete them, and doing so breaks every other user and application in that organization. Treat
> rotation as a coordinated change.

`get_token` and `rotate_token` return **404** when no M2M application exists yet — the fix is
`create_token` on the same path.

Exchange the pair for a bearer token:

```
POST https://auth.canopy.umbra.space/oauth/token
Content-Type: application/json
{"client_id":"…","client_secret":"…","audience":"https://api.canopy.umbra.space","grant_type":"client_credentials"}
```

Tokens last `expires_in` (86400s). The exchange is capped at **50 per rolling 24 hours per
client**, so cache the token and reuse it — do not mint one per call. Over the cap you get HTTP
**400** with the real 429 at `error_description.code`, plus `rate_limit_refresh` telling you when
you may try again.

## Part 2 — know what you are allowed to order

- `get_organization_settings` — your organization's settings, including default product types
- `list_all_product_constraints_for_logged_in_org` — every product constraint that applies to you
- `get_constraints_for_contract_or_org_default` — resolved constraints for a specific
  `contract_id`, an `organization_id`'s default contract, or (with neither) your own org's default

Read these before building request payloads; they tell you which imaging modes, scene sizes and
product types your contract actually permits.

## Part 3 — create the DeliveryConfig

`post_delivery_config` (POST `/delivery-config/`) creates one. Supported destinations are
customer-owned **AWS S3** and **Google Cloud Storage** buckets, with an option to zip assets
before delivery.

For the `S3_UMBRA_ROLE` type you must allow Umbra's delivery role in your bucket policy — and, if
the bucket is encrypted, in the KMS key policy too. The sandbox roles Umbra publishes are:

- AWS commercial — `arn:aws:iam::922714215458:role/prod-prod-sbx-s3-external-delivery`
- AWS GovCloud — `arn:aws-us-gov:iam::537355544329:role/prod-gov-prod-sbx-sar-data-delivery-cross-account-role`

Production role ARNs are in the delivery-configs documentation.

`list_delivery_configs` (GET `/delivery-config/`) returns all configs for your organization. It is
deliberately **unpaginated** — each org is expected to have only a few.

## Part 4 — verify before you rely on it

`verify_delivery_config` (POST `/delivery-config/{delivery_config_id}/verify`) publishes a dummy
file to your configured destination. On success the config's status becomes `ACTIVE`; on failure
it becomes `ERROR` with detail in `status_message`.

**Always verify before attaching the config to a real Task.** A misconfigured bucket policy fails
after the imagery has already been collected and billed.

## Part 5 — attach and consume

Pass the config's id as `deliveryConfigId` on `create_task`. The Task and its Collects reach
`DELIVERED` when the copy operation completes.

Call `get_schema` (GET `/collect-metadata/schema`) for the JSON Schema of the Collect metadata
delivered alongside the imagery, so you can validate what lands in your bucket.

## Deleting

`delete_delivery_config` sets the config's status to `DELETED` — a soft delete, not a hard one.

## Error handling

- **404** on `get_token`/`rotate_token` — no M2M application; call `create_token`.
- **422** — `HTTPValidationError`; read `detail[].loc` and `detail[].type`.
- **401** — token expired or wrong audience for the environment.
- **429** — 5 writes/second per organization; back off with jitter.

## Test it in the sandbox first

All DeliveryConfig operations are supported in the sandbox and fake data is delivered exactly as
in production. Use the sandbox role ARNs above in your bucket policy while testing.
