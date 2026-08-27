# AI Agent Guide — roadless-rule

## Repo relationship

This is a **client app repo** built from [geo-agent-template](https://github.com/boettiger-lab/geo-agent-template). The core library lives at [boettiger-lab/geo-agent](https://github.com/boettiger-lab/geo-agent) and is loaded from CDN — you never modify it here.

| Repo | Purpose |
|---|---|
| `geo-agent` | Core library (map, chat, agent, tools). Source of truth for all functionality. |
| `geo-agent-template` | Starter template this repo was created from. |
| `roadless-rule` | **This repo.** Config only — `index.html`, `layers-input.json`, `system-prompt.md`, `k8s/`. |

**Full docs:** [boettiger-lab.github.io/geo-agent](https://boettiger-lab.github.io/geo-agent/)
— includes the complete configuration reference, deployment guide, and agent loop internals.

The schema below is kept inline so you can work without a network fetch. If it conflicts with the docs, the docs are authoritative.

---

## What you configure (and what you don't)

**You configure:** `layers-input.json` (which datasets to show and how), `system-prompt.md` (LLM persona and guidelines), and `k8s/` manifests if deploying to Kubernetes.

**You do not write JavaScript.** The core map, chat, agent, and tool modules are loaded from the CDN. Do not create or modify JS files in a client app repo.

### Writing the system prompt

**Keep `system-prompt.md` lean.** The MCP query tool (`list_datasets`, `get_schema`) already provides the agent with dataset titles, descriptions, column schemas, coded values, and exact S3 parquet paths at runtime. Do not duplicate any of this in the system prompt — it drifts out of sync and can contradict the tools.

What **belongs** in `system-prompt.md`:
- Domain-specific context the tools cannot provide (e.g., "this dataset has one row per funding transaction, not per site — deduplicate acres before summing")
- Attribution and framing guidance (e.g., how to describe data sources to users)
- Cross-dataset pitfalls (e.g., "Dataset A uses state abbreviations, Dataset B uses full names")
- Map-vs-SQL decision guidance and interaction style
- "Data tool, not advisor" guardrails if the agent should avoid giving policy opinions

What **does not belong** in `system-prompt.md`:
- Column listings or S3 paths (use `get_schema` instead — direct the agent to call it)
- Multiple SQL examples with hardcoded paths (these go stale and may contradict the MCP tool's own query optimization rules)
- DuckDB configuration details (thread count, extensions)
- Dataset descriptions that repeat what's in the STAC catalog

Instead, add a "Discovering data" section directing the agent to verify against the dataset metadata (via `list_datasets` / `get_schema`) before writing any SQL.

---

## Branch and deployment workflow

**`main` is the live branch.** The k8s Deployment defined in `k8s/deployment.yaml` clones from `main` at pod startup — whatever is on `main` is what runs in production.

> ⚠️ **Read the Deployment name and namespace from `k8s/deployment.yaml` — never copy them from an example in this doc.** Every fork has its own name (the manifest's `metadata.name` / `metadata.namespace`). The commands below reference the manifest file directly (`-f k8s/deployment.yaml`) so you physically cannot restart the wrong app. If you must use `deployment/<name>` form, get `<name>` from the manifest first: `kubectl get -f k8s/deployment.yaml`.

Workflow for testing CDN pin updates or config changes:
1. Create a `test/` branch, make changes, verify jsDelivr serves the new SHA.
2. Merge the `test/` branch to `main` (fast-forward is fine).
3. Restart the deployment: `kubectl rollout restart -f k8s/deployment.yaml` (reads name + namespace from the manifest).

Do **not** merge to main before verifying the CDN SHA is live — jsDelivr can take up to an hour to index a new tag.

---

## Deployment

Full guide: [boettiger-lab.github.io/geo-agent/docs/guide/deployment](https://boettiger-lab.github.io/geo-agent/docs/guide/deployment)

**Read the sections below before fetching that URL** — they cover the two common k8s patterns. Fetch the docs only if you need details beyond what's here (e.g., GitHub Pages, Hugging Face Spaces, private data modules).

> **If you lack credentials or permissions** to run `kubectl` or `git push`, do not attempt to discover or work around credentials. Instead, provide the user with the exact commands to run.

### Public repo (k8s git-clone pattern)

The pod's init container clones the GitHub repo at startup. **Push to GitHub first, then restart.**

```bash
git add <files> && git commit -m "<message>" && git push
# name + namespace come from the manifest — do not hardcode them
kubectl rollout restart -f k8s/deployment.yaml
kubectl rollout status  -f k8s/deployment.yaml
```

Restarting without pushing first serves stale code.

> ⛔ **`rollout restart` never reads `k8s/deployment.yaml`.** It re-runs the Deployment object
> already in the cluster. If your change touched anything under `k8s/`, you must
> **`kubectl apply -f k8s/deployment.yaml` first** — apply updates the pod template and rolls
> automatically, so no separate restart is needed.
>
> ```bash
> git add <files> && git commit -m "<message>" && git push
> kubectl apply -f k8s/deployment.yaml    # ONLY needed if k8s/ changed — also triggers the rollout
> kubectl rollout status -f k8s/deployment.yaml
> ```
>
> **This fails silently, and the symptom looks like the app ignoring your change.** Adding a file
> the init container must copy (`docs.html`, an extra asset) is the usual way to hit it: the pod
> re-clones and picks up your `layers-input.json` edits, so the config looks deployed — but the
> in-cluster init container still has the old `cp` list, so the new file never lands in
> `/usr/share/nginx/html/`. nginx's `try_files $uri $uri/ /index.html` then answers that URL with
> the SPA shell: **HTTP 200 with the wrong page**, not a 404.
>
> Diagnose by size, not status code — a served file the same byte count as `index.html` *is* the
> fallback:
>
> ```bash
> curl -s -o /dev/null -w '%{http_code} %{size_download}\n' https://<host>/docs.html
> # Confirm what the cluster is actually running:
> kubectl get -f k8s/deployment.yaml -o jsonpath='{.spec.template.spec.initContainers[0].command}'
> ```

### Private repo (ConfigMap pattern)

When the GitHub repo is private, the pod reads content from a k8s ConfigMap instead of git-cloning. **Never edit `k8s/content-configmap.yaml` directly** — it is generated from source files.

```bash
# 1. Edit source files (index.html, layers-input.json, system-prompt.md)
# 2. Regenerate the ConfigMap
bash scripts/generate-configmap.sh
# 3. Apply and restart
kubectl apply -f k8s/content-configmap.yaml
kubectl rollout restart -f k8s/deployment.yaml
kubectl rollout status  -f k8s/deployment.yaml
# 4. Commit and push source files (not just the generated configmap)
git add <source-files> k8s/content-configmap.yaml && git commit -m "<message>" && git push
```

The git push does **not** update running pods — step 3 does. Skipping `generate-configmap.sh` and re-applying serves the old ConfigMap.

For private data modules (rclone sidecar, oauth2-proxy, private parquet credentials): [docs/guide/private-deployment](https://boettiger-lab.github.io/geo-agent/docs/guide/private-deployment)

### CDN versioning

`index.html` tracks `@main` by default:

```html
<script type="module" src="https://cdn.jsdelivr.net/gh/boettiger-lab/geo-agent@main/app/main.js"></script>
```

**When testing a geo-agent PR:** pin to the PR's HEAD commit hash, verify jsDelivr serves it, then return to `@main` when done:

```bash
# Get latest SHA from a PR
gh pr view 166 --repo boettiger-lab/geo-agent --json headRefOid --jq '.headRefOid[:8]'

# Verify jsDelivr serves it before deploying
curl -sI https://cdn.jsdelivr.net/gh/boettiger-lab/geo-agent@<sha>/app/style.css | grep HTTP
# Must return HTTP/2 200
```

Replace all three occurrences of the SHA in `index.html` (style.css, chat.css, sidebar.css, main.js), commit to `main`, and restart.

---

## Full `layers-input.json` schema

### Top-level fields

| Field | Required | Type | Description |
|---|---|---|---|
| `catalog` | Yes | string | STAC catalog root URL |
| `collections` | Yes | array | Collection specs (see below) |
| `view` | No | object | `{ "center": [lon, lat], "zoom": z }` |
| `titiler_url` | No | string | TiTiler server for COG rasters (default: `https://titiler.nrp-nautilus.io`) |
| `mcp_url` | No | string | MCP/DuckDB server URL for SQL analytics |
| `llm` | No | object | LLM config for user-provided key mode (see below) |
| `welcome` | No | object | `{ "message": "...", "examples": ["...", "..."] }` |

> **Security note:** The public MCP server (`https://duckdb-mcp.nrp-nautilus.io/mcp`) is open — no auth token is required or set. The `mcp-data-server` supports optional bearer token auth: if `MCP_AUTH_TOKEN` is set in the server's environment it enforces auth on all requests; if unset, the server is open. The active deployment does not set `MCP_AUTH_TOKEN`, so no token is needed in client apps.

### Collection-level fields

Each `collections` entry is a bare string (loads all visual assets) or an object:

| Field | Type | Description |
|---|---|---|
| `collection_id` | string | **Must exactly match the `"id"` field in the STAC collection JSON** — not a label you invent. Verify before use (see below). |
| `collection_url` | string | Direct STAC collection JSON URL — bypasses root catalog traversal |
| `group` | string | Layer toggle group label |
| `assets` | array | Asset selector (see below). Omit to load all visual assets. |
| `display_name` | string | Override collection title in UI |

### Asset config — vector / PMTiles

Each `assets` entry is a bare string (the STAC asset key) or a config object:

| Field | Type | Description |
|---|---|---|
| `id` | string | **Required.** STAC asset key (e.g., `"pmtiles"`) |
| `alias` | string | Alternative layer ID — use to create two logical layers from one STAC asset with different filters |
| `display_name` | string | Layer toggle label |
| `visible` | boolean | Default visibility (default: `false`) |
| `default_style` | object | MapLibre fill paint properties |
| `outline_style` | object | MapLibre line paint for an auto-added outline layer |
| `layer_type` | `"line"` or `"circle"` | `"line"` for LineString features; `"circle"` for Point features — see warning below |
| `default_filter` | array | MapLibre filter expression at load time |
| `tooltip_fields` | array | Property names shown on feature hover |
| `group` | string | Override collection-level group for this layer |

### Asset config — raster / COG

| Field | Type | Description |
|---|---|---|
| `id` | string | **Required.** STAC asset key |
| `display_name` | string | Layer toggle label |
| `visible` | boolean | Default visibility (default: `false`) |
| `colormap` | string | TiTiler colormap name (e.g., `"reds"`, `"viridis"`) |
| `rescale` | string | TiTiler min,max range (e.g., `"0,150"`) |
| `legend_label` | string | Legend label |
| `legend_type` | string | `"categorical"` to use STAC `classification:classes` colors |

---

## Critical: `layer_type` vs `outline_style`

**Never use `"layer_type": "line"` to draw polygon outlines.** This tells the renderer the tile features are LineString geometries. On a polygon-feature PMTiles file, it causes MapLibre to silently render nothing.

**To draw polygon boundaries without a fill**, use `outline_style` and set `fill-opacity: 0`:

```json
{
    "id": "pmtiles",
    "display_name": "District Boundaries",
    "visible": true,
    "default_style": {
        "fill-color": "#000000",
        "fill-opacity": 0
    },
    "outline_style": {
        "line-color": "#1565C0",
        "line-width": 1.5
    }
}
```

Only set `layer_type` when the tile features match the geometry type:
- `"line"` — LineString/MultiLineString features (roads, rivers, transects)
- `"circle"` — Point/MultiPoint features (observations, stations, events)

---

## Finding collection IDs and asset IDs

**Always fetch the STAC collection JSON and verify — never guess.** The `collection_id` must match the STAC `"id"` field exactly; a mismatch causes layers to silently not appear. Run this one-liner when you have the collection URL:

### Nested / hierarchical collections

Some catalog entries are **parent collections** that contain sub-collections as `"child"` links, not assets of their own. The framework only traverses **direct children of the root catalog** — it does not recurse into parent collections to find nested sub-collections.

**Symptom:** you set a `collection_id` that exists in STAC but the layer never appears. The collection is a child of a parent collection, not of the root catalog.

**Fix:** always inspect the `links` array of every collection you encounter, and use `collection_url` to point directly to the sub-collection JSON URL:

```python
import urllib.request, json
url = "<parent_collection_url>"
d = json.loads(urllib.request.urlopen(url).read())
print("id:", d["id"])
for l in d.get("links", []):
    if l.get("rel") == "child":
        print("  child:", l["href"], "|", l.get("title",""))
```

Then in `layers-input.json`, set both `collection_id` (the exact STAC `"id"`) and `collection_url` (the direct URL) so the framework bypasses root-catalog traversal:

```json
{
    "collection_id": "pad-us-4.1-fee",
    "collection_url": "https://s3-west.nrp-nautilus.io/public-padus/padus-4-1/fee/stac-collection.json",
    "assets": [...]
}
```

```bash
curl -s <collection_url> | python3 -c "
import json, sys
d = json.load(sys.stdin)
print('collection_id:', d['id'])
for k, v in d.get('assets', {}).items():
    vl = v.get('vector:layers', 'MISSING')
    print(f'  asset: {k}  type: {v.get(\"type\",\"\")}  vector:layers: {vl}')
"
```

This also checks `vector:layers` on each PMTiles asset. If it shows `MISSING`, the STAC collection needs to be patched before the layer will render — the app falls back to the asset key as the source-layer name, which is almost always wrong.

Alternatively, browse the catalog in STAC Browser:

```
https://radiantearth.github.io/stac-browser/#/external/s3-west.nrp-nautilus.io/public-data/stac/catalog.json
```

Open a collection → the collection `id` is shown at the top. Under **Assets**, the keys (e.g., `"pmtiles"`, `"v2-total-2024-cog"`) are the `id` values for asset entries. For PMTiles, the asset's `vector:layers` field lists internal layer names — the app reads this automatically, no manual config needed.

### Verifying PMTiles fields for `tooltip_fields` and `default_filter`

PMTiles tiles contain only a subset of the parquet columns — tippecanoe selects fields at tile-build time. **Do not assume field names from the STAC `table:columns` schema are available in the tiles.** Before setting `tooltip_fields` or `default_filter`, inspect the PMTiles metadata directly:

```bash
python3 -c "
import urllib.request, struct, json
url = '<pmtiles_url>'
req = urllib.request.Request(url, headers={'Range': 'bytes=0-16383'})
data = urllib.request.urlopen(req).read()
off = struct.unpack_from('<Q', data, 24)[0]
ln  = struct.unpack_from('<Q', data, 32)[0]
req2 = urllib.request.Request(url, headers={'Range': f'bytes={off}-{off+ln-1}'})
meta = json.loads(urllib.request.urlopen(req2).read())
for layer in meta.get('vector_layers', []):
    print('layer name:', layer['id'])
    print('fields:', list(layer.get('fields', {}).keys()))
"
```

The `vector_layers[].id` value is the internal layer name (must be present in `vector:layers` in the STAC asset). The `vector_layers[].fields` keys are the only field names valid for `tooltip_fields` and `default_filter`.

---

## Troubleshooting: layer not appearing in the overlay list

Two common causes:

1. **`collection_id` mismatch** — the value in `layers-input.json` does not match the STAC collection's actual `"id"` field. Run the one-liner above and compare. The framework silently drops the collection if the IDs don't match.

2. **Wrong source-layer name** — the `vector:layers` field in the STAC asset is missing or incorrect, so the app uses the asset key as the source-layer name and MapLibre finds no matching layer in the tiles. Check `vector:layers` with the one-liner above, and verify it matches the `vector_layers[].id` value from the PMTiles metadata script.

---

## MapLibre filter syntax

Use the modern `match` form for list membership:

```json
["match", ["get", "ColumnName"], ["value1", "value2"], true, false]
```

Do **not** use the legacy `["in", "ColumnName", "value1", "value2"]` form — it is silently ignored by current MapLibre.

---

## LLM config (user-provided key mode)

```json
"llm": {
    "user_provided": true,
    "default_endpoint": "https://openrouter.ai/api/v1",
    "models": [
        { "value": "anthropic/claude-sonnet-4", "label": "Claude Sonnet" },
        { "value": "google/gemini-2.5-flash",   "label": "Gemini Flash" }
    ]
}
```

Omit the `llm` block entirely for Kubernetes deployments where `config.json` is injected server-side.

---

# Repo-specific notes — roadless-rule

Everything above is the generic framework reference. This section is what an agent working on *this*
repo needs and cannot get from the tools.

## What this app is auditing

USDA's 2026 proposal to rescind the 2001 Roadless Area Conservation Rule — **91 FR 53827**
(2026-08-20), RIN 0596-AD66, docket FS-2025-0001, comments close **2026-09-21**. The proposal is not
enacted. Any copy you write must use the conditional ("would remove"), and legal status must be its
own field in a layer label, never folded into a word like "reduced".

## ⛔ The four denominators

Do not write a percentage anywhere in this repo without naming its base.

| Base | Acres | Derivation |
|---|---:|---|
| All IRA | 58,419,694 | `SUM(ACRES)` over the layer |
| **Rule-affected** | **44,701,002** | `WHERE STATE NOT IN ('ID','CO')` |
| On NFS lands | ~44,300,000 | −400k, DEIS Vol I p. 30 fn. 9 |
| **Potentially affected environment** | **40,049,537** | −wilderness / WSA / W&SR wild segments, DEIS Vol I Table 12 |

A **fifth base** answers a different question and must never be substituted into the four above:
**NFS surface ownership, 193,174,461 ac** (`nfs-surface-ownership`, `OWNERCLASS = 'USDA FOREST
SERVICE'`) — the base for "share of Forest Service land" claims. The three administrative envelopes
(`proclaimed-forest` 225.1M, `administrative-forest` 236.8M, `ranger-district` 237.1M) are **not**
ownership; using one inflates the denominator by up to 22.6%.

Idaho 9,285,370 + Colorado 4,433,322 = 13,718,692. West-10 = **95.61%** of the rule-affected base,
**73.16%** of the all-IRA base. Verify any figure you add:

```sql
SELECT SUM(ACRES) FROM read_parquet('s3://public-usfs/roadless-areas-2001.parquet')
WHERE STATE NOT IN ('ID','CO');   -- 44,701,002
```

## ⚠️ The `CATEGORY` trap

`CATEGORY` (`1B` / `1B-1` / `1C`) records **pre-rule forest-plan direction**, not rule status. `1C`
reads as though road construction were allowed; it was not. Never style, filter, or label as though
this column carried protection level.

## Sidebar group order is meaningful — and is set by array order

The framework has **no group-ordering field.** `map-manager.js` groups layers into an
insertion-ordered `Map`, so a group's position in the panel is the position of the **first
collection in `layers-input.json` that names it**, and a layer's position within a group is its
global insertion order. Reordering the panel means reordering the `collections` array — nothing
else. Likewise `groupCollapsed` is read off the group's **first** member only, so when a group spans
several collections, only the first one's `collapsed` flag has any effect (keep them consistent
anyway).

The eight groups are ordered by how directly each bears on the proposal, and the order carries
argument, so preserve it when adding a layer:

| # | Group | Why here |
|---:|---|---|
| 1 | Roadless areas · USFS 2001 | the subject — **the only group expanded at open** |
| 2 | Roads | what 36 CFR 294 actually regulates |
| 3 | Trails & recreation access | "roadless" ≠ inaccessible |
| 4 | Fire history | measured record: ignitions, perimeters, severity |
| 5 | Fire risk & fuels | modelled hazard, stand condition, treatment |
| 6 | National Forest System extent | the land base (5th denominator) |
| 7 | Existing protections | the Table 12 deduction → 40.0M base |
| 8 | Land cover & modification | background |

⛔ **Do not merge Roads into Trails or vice versa**, and do not restore a shared "access" label for
the two. 36 CFR 294.11 defines a road as a motor vehicle travelway over 50 inches wide; a shared
group name is what invites trail mileage into a road figure. Likewise keep *Fire history* apart from
*Fire risk & fuels* — one is a measured record, the other a model projection, and the proposal's
rationale turns on not conflating them.

⚠️ Within a group, a layer that is a **subset** of another must follow it (large-fire ignitions after
all ignitions; TIGER highways after all motor-vehicle roads; NTS routes after the agency trail
layers). The subset relationships are stated in the layer labels; the ordering is what makes them
readable at a glance.

⚠️ Four surfaces carry the group names and must be updated together, or they drift:
`layers-input.json`, `system-prompt.md` (the *What this app has* inventory), `docs.html` (the
*Datasets in this app* `<h3>`s) and `README.md` (*Layer organization*), with `DATA-SOURCES.md`
mirroring the same section order. They had already drifted into four different orderings once.

## The two roadless layers are one STAC asset

`roadless-areas-2001-pmtiles` appears **twice** in `layers-input.json`, distinguished by `alias`
(`ira-rule-affected`, `ira-idaho-colorado`) and complementary `default_filter`s on `STATE`. This is
the documented mechanism for two logical layers off one asset — the framework shares a single
MapLibre source between aliases. If you add a third view of the same data, add another alias; do not
duplicate the collection entry.

Filters use the modern `match` form (`["match", ["get","STATE"], ["ID","CO"], false, true]`). `STATE`
is a clean two-letter code and **is** present in the tiles — verified, not assumed.

## ⚠️ Raster legends cannot be overridden in config

For a **raster** asset the app derives both the legend and the TiTiler colormap from the STAC
`classification:classes` (`dataset-catalog.js:456`), and ignores `legend_classes` — the vector paths
honour it, the raster path does not. Two consequences:

- A junk or fill class in the STAC will show in the legend, and will **paint** unless its value equals
  the band `nodata`. `landfire-2024-vcc` has two such codes; see
  [data-workflows#628](https://github.com/boettiger-lab/data-workflows/issues/628).
- The colormap is inlined into the tile URL with no cap, so class count drives URL size. Roughly 945 B
  at 14 classes, **47 KB at 832** — which the tile server refuses outright. That is why LANDFIRE EVT
  has no map layer. Check the class count before adding any categorical raster.

## ⚠️ The NFS extent layers carry no state column

None of `nfs-surface-ownership`, `proclaimed-forest`, `administrative-forest`, `ranger-district` has a
state field — `REGION` is a USFS region, not a state. Any per-state figure needs a hex or clipped
spatial join (`census-2024/state` is native h8; these layers are native h10 with h9/h8/h0 parents). A
plain `ST_Intersects` **sum is wrong** — it credits a parcel's whole acreage to every state its
geometry touches, overstating Montana by roughly 9M acres.

## Layers pending ingest

Tracked in [boettiger-lab/data-workflows](https://github.com/boettiger-lab/data-workflows) under the
`roadless` label, coordinated by [#594](https://github.com/boettiger-lab/data-workflows/issues/594).
Adding each one is a `layers-input.json` entry plus a `DATA-SOURCES.md` row.

**Already added:** #585 (USFS extent — `nfs-surface-ownership` filtered to `OWNERCLASS = 'USDA
FOREST SERVICE'`, plus proclaimed / administrative / ranger-district as outline-only envelopes; Gov.
Gianforte's "nearly 60% of Forest Service land" statement is now testable — it does **not** reconcile,
the measured share being ~37%, and it is about constrained management, not roadless acreage), #586 (WHP v2023 — four collections; the *continuous* pair is mapped, the
classified pair is SQL-only because its COGs carry no `classification:classes`, so geo-agent would
paint Non-burnable and Water as top hazard), #593 (MTBS — perimeters split into wildfire and
prescribed-fire aliases, plus per-domain annual severity behind a `versions` year selector),
#587 (FPA-FOD — 2.66M ignition points as two `circle` layers coloured by cause) and #588 (roads —
`roadcore-fs` split on `OPER_MAINT_LEVEL`, plus `census-2025/roads` split on `MTFCC` into
motor-vehicle / highways / excluded-paths). Claims 1 and 3 are now testable. Note the builds
published to S3 while PRs #605, #612, #614 and #621 were still open — **STAC liveness, not merge
status, is what determines whether a layer can be added.**

⛔ **`roadcore-fs` and `census-2025/roads` overlap and must never be unioned by addition.** Measured
on a 0.3° window over the Bitterroot NF, 97 of 112 RoadCore segments (87%) lie within 25 m of a TIGER
line — Forest Service roads reappear in TIGER as `S1500` or `S1740`. Combine the two only as distance
to the *nearest* road across both. TIGER also carries no length column, so its layers are labelled in
segments while RoadCore's are labelled in miles; the two units are not comparable.

**Added from outside the `roadless` issue set:** `federal-trails-2026` (USFS NFST + NPS + BLM GTLF,
127,619 segments) and `american-rivers-nri-2024` (4,496 NPS-inventoried river segments), both in a
*Trails & recreation access* group. They answer the "roadless means inaccessible" inference that a
roads-only map invites. Neither came from a `roadless`-labelled issue — trails landed under the
unlabelled [#167](https://github.com/boettiger-lab/data-workflows/issues/167), closed long before
this app existed. **The `roadless` label is a work queue, not an inventory of what is relevant** —
browse the catalog itself before concluding nothing new is addable.

⛔ **`federal-trails-2026`'s NTS layer is a subset of its two agency layers**, and a trail is never a
road under 36 CFR 294.11 — no trail mileage may enter a road-proximity figure. ⚠️ Only
`admin_agency` and `nts_designation` are harmonized; `trail_type` (41 raw values, 31 case-folded),
`trail_class` (20) and `trail_surface` (62) carry each agency's raw vocabulary despite the
collection description claiming a harmonized schema.

⛔ **NRI is an inventory, not a protection, and is not the PAD-US W&SR stratum.** `nri-2024` records
segments *potentially eligible* for Wild & Scenic designation; the Table 12 deduction stratum comes
from PAD-US `Des_Tp`. Two datasets, two questions — never combined. `american-rivers-wild-scenic-designated`
(1,052 segments) was deliberately **not** added for exactly this reason: it is a second rendering of
the designated inventory PAD-US already supplies, the same trap as PAD-US `Des_Tp = 'IRA'`.

⚠️ **FPA-FOD was built as the 7th edition (1992–2024), not the 6th (1992–2020) that #587 was filed
against.** The 7th backfills previously underrepresented states, so pre-2021 counts differ — the
dataset id is `fpa-fod-1992-2024`.

Checked against live STAC on 2026-08-27 — none of the following has a `stac-collection.json` yet, so
none can be added. `public-landfire/landfire-2024-evc/` and `.../landfire-2024-fbfm40/` hold COGs but
**no collection JSON** (#623). A bucket prefix is not an ingest — probe for `stac-collection.json`
before believing a dataset is available.

✅ **#592 landed and is now in the app.** `wrc-2-rps-conus` and `wrc-2-rps-ak` published
2026-08-27 (521.0 M and 120.3 M res-10 cells), verified by `verify-stac.py` with data checks. Note
what the earlier note above got wrong: `public-fire` held WRC v2 "only under `raw/`" at the time,
which was true, but the layer that is mapped is the **COG at the bucket root**
(`wrc-2-rps-conus-cog.tif`), not anything under the dataset prefix — the prefix holds the hex the
assistant queries. #611 and #627 are still raw-only and remain unaddable.

| Issue | Dataset | Expected bucket |
|---|---|---|
| #590 | LANDFIRE 2023/2024 — FRCC, EVC, FBFM40 (VCC done, EVT unusable) | `public-landfire` |
| #591 | USFS Insect & Disease Detection Survey | `public-usfs` |
| #603 | TWIG interagency fuel treatments + intersections | `public-fire` |
| #606 | TNC Resilient & Connected Network — national | TBD |
| #609 | Mesic Analysis Platform — mesic persistence, valley bottoms | TBD |
| #610 | USGS INHABIT v4 — fire-promoting invasive plants | TBD |
| #611 | Wildfire Risk to Communities v2 — populated areas | `public-fire` |
| #623 | LANDFIRE 2024 — FBFM40 hex + the decoded-EVC question | `public-landfire` |
| #627 | WRC v2 — BP, CFL, Exposure (raw staged, no COG) | `public-fire` |

⚠️ **`facts-common-attributes-2026-06` and TWIG (#603) must never be unioned** — TWIG's 737,013
FACTS-CA rows overlap ours, so a union double-counts USFS treatments.

## Deployment quirks

- Namespace `schmidtdse`, slug `roadless-rule`, host `roadless-rule.nrp-nautilus.io`.
- The init container clones `main` at pod start: **push, then**
  `kubectl -n schmidtdse rollout restart deployment/roadless-rule`. Neither step works alone.
- ⛔ **If you changed `k8s/` — including adding a file to the init container's `cp` list —
  `rollout restart` is not enough.** Run `kubectl apply -f k8s/deployment.yaml` instead; it rolls
  on its own. Skipping it serves a 200 with the wrong content, not an error. See
  *Public repo (k8s git-clone pattern)* above. The four files the init container copies are
  `index.html`, `docs.html`, `layers-input.json` and `system-prompt.md` — **a new top-level file
  is invisible in production until it is added there and the manifest is applied.**
- `kubectl` needs `/home/jovyan/bin` on `PATH` — the kubeconfig's exec credential plugin shells out
  to `kubectl` itself, so a bare invocation fails with "executable kubectl not found".
- The nginx sidecar config carries two fixes that must not be "cleaned up":
  `resolver 10.96.0.10 ipv6=off` with a variabled `proxy_pass` (IPv4-only, geo-agent-ops#64) and
  `worker_processes 2` (OOM on high-core nodes, geo-agent-ops#49).
