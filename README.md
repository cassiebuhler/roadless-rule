# roadless-rule

An AI-powered interactive map for testing the numbers behind **USDA's 2026 proposal to rescind the
2001 Roadless Area Conservation Rule** (36 CFR 294, subpart B) — 91 FR 53827, published
2026-08-20, comments close **2026-09-21**.

The agency's announcement quotes several figures in support of the proposal. Almost every one is a
percentage, and almost every percentage depends on **which acreage base it is computed against** —
of which there are four, differing by 18 million acres. This app holds the underlying data so those
figures can be reproduced rather than taken on trust.

Built on the [geo-agent / GLEN](https://github.com/boettiger-lab/geo-agent) framework. **No
JavaScript to write** — the map, chat, agent and tools load from CDN; this repo is three config
files plus deployment manifests.

**Live:** <https://roadless-rule.nrp-nautilus.io> · **Framework docs:**
[boettiger-lab.github.io/geo-agent](https://boettiger-lab.github.io/geo-agent/)

## Files

```
index.html          ← HTML shell — loads GLEN core (pinned @v3.27.0) + libs from CDN
layers-input.json   ← datasets, grouping, map view, LLM settings
system-prompt.md    ← rescission-analyst persona, denominators, guardrails
DATA-SOURCES.md     ← publisher, coverage, vintage and license for every layer
k8s/                ← Kubernetes deployment (app: roadless-rule)
AGENTS.md           ← configuration reference for AI coding agents
```

## The four denominators

This is the thing the app exists for. Agency percentages are computed against the **potentially
affected environment**, not total roadless acreage:

| Base | Acres | Notes |
|---|---:|---|
| All inventoried roadless areas | 58,419,694 | 11,391 polygons, 38 states + Puerto Rico |
| **Rule-affected** | **44,701,002** | less Idaho (9,285,370) + Colorado (4,433,322), which have their own state rules |
| Rule-affected on NFS lands | ~44,300,000 | less ~400k ac of ownership-change slivers |
| **Potentially affected environment** | **40,049,537** | less designated wilderness, WSAs and Wild & Scenic wild segments — DEIS Vol I Table 12 |

"More than 44 million acres" ✅ and "more than 95% in 10 Western states" ✅ (95.61% of the
rule-affected base; 73.16% of the all-IRA base) both reconcile against this layer. The Montana
"nearly 60 percent of Forest Service land" claim does **not** — see
[DATA-SOURCES.md](DATA-SOURCES.md).

The 58.4M ↔ 44.7M split has no attribute in the data; it is `STATE NOT IN ('ID','CO')`.

## Layer organization

Layers are grouped by **what the data describes**, not by which agency publishes it. Every label
follows one form — **`what it is · PUBLISHER vintage`** — and legal status is its own field, so
"PROPOSED for rescission" reads as a status rather than an accomplished fact.

| Group | Layers |
|---|---|
| **Roadless areas · USFS 2001** | `Rule-affected · 44.7M ac — PROPOSED for rescission` · `Idaho & Colorado · 13.7M ac — state rules, excluded` |
| **Comparison strata** | `Designated wilderness · PAD-US 4.1` · `All protected areas · PAD-US 4.1` |
| **Fuels & fire** | `Completed treatments, FY2014+ · USFS FACTS 2026` · `Fire perimeters 1835–2020 · USGS 2021` · `Wildland-urban interface · SILVIS 2020` |
| **Land cover & modification** | `Land cover · NLCD 2024 (CONUS only)` · `Human modification · Theobald 2016` |

The two roadless layers are one dataset filtered two ways, via the `alias` mechanism — the map opens
with both on, so the affected area and the untouched Idaho/Colorado comparison group read as a
single picture. Every other group starts collapsed and off.

Full provenance — publisher, coverage, vintage, license, and every caveat that changes a number — is
in **[DATA-SOURCES.md](DATA-SOURCES.md)**, which the app links as "About" in its footer.

## Datasets still to come

Ingest is tracked in [boettiger-lab/data-workflows](https://github.com/boettiger-lab/data-workflows)
under the [`roadless`](https://github.com/boettiger-lab/data-workflows/issues?q=is%3Aissue+label%3Aroadless)
label, coordinated by [#594](https://github.com/boettiger-lab/data-workflows/issues/594). Two of the
three headline claims cannot be tested until they land: **Wildfire Hazard Potential**
([#586](https://github.com/boettiger-lab/data-workflows/issues/586)) for the ">40% high hazard"
claim, and **RoadCore + TIGER roads**
([#588](https://github.com/boettiger-lab/data-workflows/issues/588)) for "11.3M acres near roads".
The full table is in [DATA-SOURCES.md](DATA-SOURCES.md).

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
