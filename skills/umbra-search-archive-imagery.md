---
name: Search Umbra's SAR archive and retrieve imagery
description: >-
  Search Umbra's STAC-compliant catalogs for existing SAR imagery over an area and time range, and
  resolve items to downloadable signed asset URLs. Covers both the Archive Catalog and the STAC API
  v2 over your own tasked Collects.
api: openapi/umbra-stac-archive-openapi.yml
operations:
  - Search_search_post
  - Search_search_get
  - Get_Collections_collections_get
  - Get_Collection_collections__collection_id__get
  - Get_ItemCollection_collections__collection_id__items_get
  - Get_Item_collections__collection_id__items__item_id__get
  - get_thumbnail
  - Landing_Page__get
generated: '2026-08-05'
method: generated
source: >-
  openapi/umbra-stac-archive-openapi.yml + openapi/umbra-stac-api-v2-openapi.yml +
  https://docs.canopy.umbra.space/docs/archive-catalog-searching-via-stac-api
---

# Search Umbra's SAR archive and retrieve imagery

Two STAC surfaces, same shape, different content:

| Surface | Base URL | Contains |
|---|---|---|
| Archive Catalog | `https://api.canopy.umbra.space/archive` | Umbra's catalog of previously collected imagery |
| STAC API v2 | `https://api.canopy.umbra.space/v2/stac` | the Collects **your organization** tasked |

Both implement the STAC API Item Search Specification and the STAC API Filter Extension.

> The Archive Catalog **requires authentication** as of 2024-12-20 and is subject to the standard
> Canopy rate limits. It was previously open; older examples that omit the token are stale.

## Authenticate

`Authorization: Bearer <access_token>` on every request — same token as the tasking flow. See
`umbra-task-new-collection.md` for how to mint one.

## Step 1 — find the collections

Call `Get_Collections_collections_get` to list collections. On the Archive Catalog the documented
collection is `umbra-sar`; on the STAC API v2 the collection id is your organization. Use
`Get_Collection_collections__collection_id__get` for one collection's detail.

`Landing_Page__get` (STAC API v2 only) returns the STAC landing document with conformance classes.

## Step 2 — search

**Simple search** — `Search_search_get`, query-string parameters, good for quick lookups.

**Advanced search** — `Search_search_post`, full STAC Item Search body with the Filter Extension
(CQL2). Use this for anything real:

- `collections` — e.g. `["umbra-sar"]`
- `intersects` — a GeoJSON geometry (build one at <https://geojson.io/>)
- `datetime` — an interval such as `2020-01-01/2024-12-31`
- `limit` — page size
- `filter` — CQL2 over item properties

> **Signed URLs are conditional.** STAC item `assets` include signed download URLs only when the
> number of items returned is below a threshold (default **5**). A broad search returns items with
> empty or unsigned assets. Narrow the query, or re-fetch the single item with
> `Get_Item_collections__collection_id__items__item_id__get` to get its signed assets.

## Step 3 — page through results

Pagination is token-cursor, not offset. Follow the `rel="next"` entry in the response `links`
array, passing the opaque `token` with your `limit`. The `context` object reports `limit` and
`returned`. `Get_ItemCollection_collections__collection_id__items_get` lists all items in a
collection with the same paging.

## Step 4 — read the item

Umbra items carry namespaced properties alongside the STAC core: `umbra:status`,
`sar:instrument_mode` (e.g. `SPOTLIGHT`), `platform`, `datetime`, `created`. The vocabulary is
published as a STAC extension at <https://github.com/Umbra-Space/umbra-stac-extension>.

For a quick visual, `get_thumbnail` (Archive Catalog) returns a thumbnail by `archive_id`, and the
Tiles API `preview_cog_preview_get` renders a preview of a COG dataset.

## Joining from a Task you submitted

A Collect's `archiveIds` field is the bridge: task a collection, poll to `PROCESSED`, then use
those ids against these STAC operations to fetch the imagery.

## Third-party tooling Umbra recommends

Umbra ships no SDK. Its own developer-tooling page points at `pystac-client`:

```python
import pystac_client
catalog = pystac_client.Client.open(
    "https://api.canopy.umbra.space/archive/",
    headers={"Authorization": f"Bearer {CANOPY_TOKEN}"},
)
items = catalog.search(
    max_items=10, collections=["umbra-sar"],
    intersects=intersect_geometry, datetime="2020-01-01/2024-12-31",
).item_collection()
```

`leafmap` is recommended for drawing search bounds interactively in Jupyter.

## Error handling

- **422** — `HTTPValidationError`; read `detail[].loc` and `detail[].type`. Malformed geometry and
  bad datetime intervals land here.
- **401** — token missing/expired, or minted for the wrong environment audience.
- **429** — 25 reads/second per organization. Back off with jitter.

## Licensing

Umbra sells its imagery under **CC BY 4.0** and runs an open data program at
<https://umbra.space/open-data/> — worth checking before paying for an archive order.
