# roadless-rule

An AI-powered interactive map for testing the numbers behind **USDA's 2026 proposal to rescind the
2001 Roadless Area Conservation Rule** (36 CFR 294, subpart B) — 91 FR 53827, published
2026-08-20, comments close **2026-09-21**.

The agency's announcement quotes several figures in support of the proposal. Almost every one is a
percentage, and almost every percentage depends on **which acreage base it is computed against** —
of which there are four, differing by 18 million acres. This app holds the underlying data so those
figures can be reproduced rather than taken on trust.

Built on the [geo-agent / GLEN](https://github.com/boettiger-lab/geo-agent) framework. **No
JavaScript to write** — the map, chat, agent and tools load from CDN; this repo is config files, a
static About page, and deployment manifests.

**Live:** <https://roadless-rule.nrp-nautilus.io> · **Framework docs:**
[boettiger-lab.github.io/geo-agent](https://boettiger-lab.github.io/geo-agent/)

## Files

```
index.html          ← HTML shell — loads GLEN core (pinned @v3.27.0) + libs from CDN
docs.html           ← "About" page served at /docs.html — sources, denominators, claims
layers-input.json   ← datasets, grouping, map view, LLM settings
system-prompt.md    ← rescission-analyst persona, denominators, guardrails
DATA-SOURCES.md     ← publisher, coverage, vintage and license for every layer
k8s/                ← Kubernetes deployment (app: roadless-rule)
AGENTS.md           ← configuration reference for AI coding agents
```

## The five denominators

This is the thing the app exists for. Agency percentages are computed against the **potentially
affected environment**, not total roadless acreage:

| Base | Acres | Notes |
|---|---:|---|
| All inventoried roadless areas | 58,419,694 | 11,391 polygons, 38 states + Puerto Rico |
| **Rule-affected** | **44,701,002** | less Idaho (9,285,370) + Colorado (4,433,322), which have their own state rules |
| Rule-affected on NFS lands | ~44,300,000 | less ~400k ac of ownership-change slivers |
| **Potentially affected environment** | **40,049,537** | less designated wilderness, WSAs and Wild & Scenic wild segments — DEIS Vol I Table 12 |
| **All NFS land** *(a different question)* | **193,174,461** | Forest Service **surface ownership** — the base for "share of Forest Service land" claims, never for "share of roadless area" |

"More than 44 million acres" ✅ and "more than 95% in 10 Western states" ✅ (95.61% of the
rule-affected base; 73.16% of the all-IRA base) both reconcile against this layer. The Montana
"nearly 60 percent of Forest Service land" claim does **not**: against Montana surface ownership the
share is roughly 37%, and no Forest Service extent layer is small enough to yield 60% — see
[DATA-SOURCES.md](DATA-SOURCES.md).

⚠️ The fifth base is **surface ownership**, not one of the three administrative envelopes the Forest
Service also publishes (proclaimed 225.1M, administrative 236.8M, ranger districts 237.1M). Those
enclose inholdings the agency does not own; substituting one inflates the denominator by up to 22.6%.

The 58.4M ↔ 44.7M split has no attribute in the data; it is `STATE NOT IN ('ID','CO')`.

## Layer organization

Layers are grouped by **what the data describes**, not by which agency publishes it. Every label
follows one form — **`what it is · PUBLISHER vintage`** — and legal status is its own field, so
"PROPOSED for rescission" reads as a status rather than an accomplished fact.

Groups run in order of how directly they bear on the proposal: the subject, then what the rule
regulates, then what "roadless" does *not* mean, then the agency's stated rationale, then the
denominators, then background. Within a group, a layer that is a **subset** of another always
follows it.

| # | Group | Layers |
|---:|---|---|
| 1 | **Roadless areas · USFS 2001** | `Rule-affected · 44.7M ac — PROPOSED for rescission` · `Idaho & Colorado · 13.7M ac — state rules, excluded` |
| 2 | **Roads** | `NFS roads open to vehicles, 263,807 mi · USFS 2025` · `NFS roads closed & stored (ML1), 103,945 mi · USFS 2025` · `All motor-vehicle roads, 16,470,232 segments · TIGER 2025` · `Highways & secondary roads, 268,817 segments · TIGER 2025` · `Walkways & paths, 20,667 segments — NOT roads under 36 CFR 294.11 · TIGER 2025` |
| 3 | **Trails & recreation access** | `USFS trails, 134,983 mi · Federal Trails 2026` · `NPS & BLM trails, 25,916 mi · Federal Trails 2026` · `National Trails System routes, 12,488 mi · Federal Trails 2026` · `Rivers with outstanding values, 90,476 mi · NPS NRI 2024` |
| 4 | **Fire history** | `Ignitions by cause 1992–2024 · FPA-FOD` · `Large-fire ignitions ≥1,000 ac 1992–2024 · FPA-FOD` · `Wildfire perimeters 1984–2024 · MTBS` · `Prescribed fire perimeters 1984–2024 · MTBS` · `Burn severity by year · MTBS (CONUS)` · `(Alaska)` · `Fire perimeters 1835–2020 · USGS 2021` |
| 5 | **Fire risk & fuels** | `Wildfire hazard index · WHP 2023 (CONUS)` · `(Alaska)` · `Vegetation condition class · LANDFIRE 2024 (CONUS only)` · `Completed treatments, FY2014+ · USFS FACTS 2026` · `Wildland-urban interface · SILVIS 2020` |
| 6 | **National Forest System extent** | `Forest Service ownership, 193.2M ac · USFS 2025` · `Proclaimed boundary, 225.1M ac · USFS 2025` · `Administrative boundary, 236.8M ac · USFS 2025` · `Ranger districts, 237.1M ac · USFS 2025` |
| 7 | **Existing protections · PAD-US 4.1** | `Designated wilderness · PAD-US 4.1` · `Wilderness study areas · PAD-US 4.1` · `Wild & Scenic Rivers, wild segments · PAD-US 4.1` |
| 8 | **Land cover & modification** | `Land cover · NLCD 2024 (CONUS only)` · `Human modification · Theobald 2016` |

**Roads** and **trails** are deliberately separate groups, not one "access" group: 36 CFR 294.11
defines a road as a motor vehicle travelway over 50 inches wide, so no trail mileage may enter a
road figure. Splitting **fire** into *history* (what burned, where ignitions start) and *risk &
fuels* (forward-looking hazard, stand condition, treatment) keeps a measured record apart from a
modelled projection.

Only `Forest Service ownership` in the NFS extent group is ownership; the other three are
administrative envelopes, drawn as outlines with no fill so the distinction is visible rather than
merely documented. The NFS extent and PAD-US groups sit together because they are the denominator
machinery: the land base, and the Table 12 deduction taken out of it.

The two roadless layers are one dataset filtered two ways, via the `alias` mechanism. Only
`Rule-affected` is on at open, so the default map shows the 44.7M base — switch `Idaho & Colorado`
on whenever the question is comparative. Groups 1 and 2 are expanded by default; everything below
them starts collapsed, and every layer outside the roadless group starts switched off.

Full provenance — publisher, coverage, vintage, license, and every caveat that changes a number — is
in **[DATA-SOURCES.md](DATA-SOURCES.md)**. The app's "About" footer link points to
**[docs.html](docs.html)**, a self-hosted page served at `/docs.html` alongside the map — it is
derived from DATA-SOURCES.md, so the two must be updated together.

## Datasets still to come

Ingest is tracked in [boettiger-lab/data-workflows](https://github.com/boettiger-lab/data-workflows)
under the [`roadless`](https://github.com/boettiger-lab/data-workflows/issues?q=is%3Aissue+label%3Aroadless)
label, coordinated by [#594](https://github.com/boettiger-lab/data-workflows/issues/594). All three
headline claims are now testable — **Wildfire Hazard Potential**
([#586](https://github.com/boettiger-lab/data-workflows/issues/586)) and **RoadCore + TIGER roads**
([#588](https://github.com/boettiger-lab/data-workflows/issues/588)) have both landed. What is still
in flight, and what is blocked, is in the full table in [DATA-SOURCES.md](DATA-SOURCES.md).

## Local development

```bash
python -m http.server 8000
# Open http://localhost:8000 — enter your API key in the ⚙ settings panel
```

## Deployment

Kubernetes on NRP Nautilus, namespace `schmidtdse`. The pod's init container clones `main` at
startup, so **push, then restart** — pushing alone changes nothing and restarting alone serves
stale config.

```bash
kubectl apply -f k8s/                                        # first time
kubectl -n schmidtdse rollout restart deployment/roadless-rule   # after every push
kubectl -n schmidtdse rollout status  deployment/roadless-rule
```

The LLM key is injected server-side by the nginx sidecar (`/api/llm` reverse proxy, `PROXY_KEY` from
the `open-llm-proxy-secrets` secret) — the browser never holds it. The `llm` block in
`layers-input.json` is what makes the local `http.server` flow work; in-cluster the injected
`config.json` takes precedence.
