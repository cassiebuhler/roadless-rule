# Data sources

Every layer in this app comes from a published government or academic dataset. Nothing here is
modeled, estimated, or synthesized by the app — the only processing is format conversion
(GeoParquet / PMTiles / H3 hex) and, where noted, a filter applied at display time.

All layers are served from the public STAC catalog at
[`s3-west.nrp-nautilus.io/public-data/stac/catalog.json`](https://s3-west.nrp-nautilus.io/public-data/stac/catalog.json)
and processed by the [Boettiger Lab](https://github.com/boettiger-lab) `cng-datasets` pipeline. The
catalog entry for each layer carries the full column schema, feature counts and provenance; the
assistant reads it at runtime, so you can also just ask it "where does this layer come from?".

The sidebar is organized by **what the data describes**, not by which agency publishes it.

## How to read a layer label

Every layer label follows one form: **`what it is · PUBLISHER vintage`** — for example
`Fire perimeters 1835–2020 · USGS 2021`. The trailing token is either a **year** or, where the source
publishes numbered releases, a **version** (`PAD-US 4.1` is the only one here).

Every dataset in this app is a **fixed vintage** — a published release or a frozen snapshot, not a
live service. Where a layer's label carries a filter (`FY2014+`, `CONUS only`), that filter is applied
in the map only; the underlying parquet is complete and the assistant can query all of it.

## Publishers

| Acronym | Full name |
|---|---|
| **USFS** | USDA Forest Service |
| **USGS** | U.S. Geological Survey |
| **MRLC** | Multi-Resolution Land Characteristics Consortium (USGS EROS) |
| **PAD-US** | Protected Areas Database of the United States (USGS Gap Analysis Project) |
| **FACTS** | Forest Service Activity Tracking System |
| **SILVIS** | SILVIS Lab, University of Wisconsin–Madison |

## Layers

### Roadless areas · USFS 2001

| Field | Value |
|---|---|
| Layers | `Rule-affected · 44.7M ac — PROPOSED for rescission`, `Idaho & Colorado · 13.7M ac — state rules, excluded` |
| Publisher | USDA Forest Service, Geospatial Service and Technology Center |
| Coverage | National — 38 states + Puerto Rico, incl. Alaska |
| Vintage | 2001 (the inventory designated by the 2001 rule) |
| Features | 11,391 polygons · 58,419,694 acres |
| License | Public domain |
| Source | [`S_USA.RoadlessArea_2001.zip`](https://data.fs.usda.gov/geodata/edw/edw_resources/shp/S_USA.RoadlessArea_2001.zip) |
| Collection | `roadless-areas-2001` |

Both map layers are the **same dataset**, split on `STATE`: the rule-affected layer is
`STATE NOT IN ('ID','CO')`, the other its complement. There is no attribute recording which rule
governs an area.

⚠️ **`CATEGORY` (`1B` / `1B-1` / `1C`) records forest-plan direction from *before* the rule took
effect.** `1C` reads as though road construction were allowed; it was not. The 2001 rule prohibits
road construction across every inventoried roadless area regardless of category.

⚠️ **Idaho and Colorado boundaries here are the superseded 2001 ones.** Those states adopted their own
roadless rules in 2008 and 2012; the Forest Service publishes the operative boundaries as separate
datasets, not included here.

USFS cautions that source scales vary across this layer and that boundaries cannot be expected to
align with features from other datasets — the National Forest Planning Record documents remain the
official inventory. Treat adjacency and buffer-distance results as approximate.

### Comparison strata

| Layer | Publisher | Coverage | Vintage | License | Collection |
|---|---|---|---|---|---|
| `Designated wilderness · PAD-US 4.1` | USGS Gap Analysis Project | National | 4.1 (released Mar 2025) | Public domain | `pad-us-4.1-combined` |
| `All protected areas · PAD-US 4.1` | USGS Gap Analysis Project | National | 4.1 | Public domain | `pad-us-4.1-combined` |

Both are the PAD-US **combined** layer; the wilderness layer is filtered to `Des_Tp = 'WA'` (995
features, 113.8M ac). DOI [10.5066/P96WBCHS](https://doi.org/10.5066/P96WBCHS).

⚠️ **PAD-US combined contains overlapping polygons** for the same ground (separate fee, easement,
proclamation and management rows), so `SUM(GIS_Acres)` over it is meaningless at national scale —
deduplicate by unit or intersect geometries first.

⚠️ **PAD-US carries its own `Des_Tp = 'IRA'` records** (~58.2M ac, close to the USFS 58.4M). It is a
different rendering of the same inventory, not an independent one — never add it to the USFS roadless
layer.

### Fuels & fire

| Layer | Publisher | Coverage | Vintage | License | Collection |
|---|---|---|---|---|---|
| `Completed treatments, FY2014+ · USFS FACTS 2026` | USDA Forest Service | National (NFS lands) | 2026-06 snapshot | Public domain | `facts-common-attributes-2026-06` |
| `Fire perimeters 1835–2020 · USGS 2021` | U.S. Geological Survey | National | 2021 release | Public domain | `usgs-fires-2021-combined` |
| `Wildland-urban interface · SILVIS 2020` | SILVIS Lab, UW–Madison | National (census blocks) | 2020 (v4, decades 1990–2020) | CC-BY-4.0 | `silvis-wui-2020` |

⚠️ **The FACTS layer is not "hazardous fuels treatments."** It is filtered to
`FISCAL_YEAR_COMPLETED >= 2014` across **all** completed activity types. The agency's 5% claim uses an
undisclosed subset (see below). `NFPORS_CATEGORY` is null on the majority of rows and is not a usable
fuels selector. Fiscal years, not calendar.

⚠️ **The fire layer is not an annual fire history.** USGS combines many source datasets across
1835–2020 with very uneven completeness by era and region; counts collapse before ~1980.
`Overlap_Within_1_or_2_Flag` and `Exclude_From_Summary_Rasters` mark records USGS itself says not to
use in area summaries.

⚠️ **This is the SILVIS WUI, not the HFRA WUI** the Draft EIS uses for its 9.8M-acre figure — different
definitions, not interchangeable. `WUIFLAG2020`: `0` non-WUI, `1` intermix, `2` interface; the map is
filtered to `> 0`.

### Land cover & modification

| Layer | Publisher | Coverage | Vintage | License | Collection |
|---|---|---|---|---|---|
| `Land cover · NLCD 2024 (CONUS only)` | MRLC / USGS EROS | **CONUS only** | 2024 (Annual NLCD Collection 1.2) | Public domain | `nlcd` |
| `Human modification · Theobald 2016` | Theobald et al. | Global | circa 2016, ~1 km | CC-BY-4.0 | `global-human-modification` |

⚠️ **NLCD has no Alaska**, which holds 14,778,681 roadless acres — 25% of the all-IRA total and the
largest single state. Any land-cover share computed from it silently drops Alaska. Classified raster:
use `mode`, not `mean`; nodata `250`.

Human modification is a continuous 0–1 index (0 = unmodified, 1 = fully modified) — average it, never
sum it. DOI [10.6084/m9.figshare.7283087](https://doi.org/10.6084/m9.figshare.7283087).

## Reproducing the agency's numbers

The rulemaking under audit: **91 FR 53827** (published 2026-08-20), RIN **0596-AD66**, docket
**FS-2025-0001**, action *remove and reserve 36 CFR part 294, subpart B*. **Comments close
2026-09-21.** Notice of intent: 90 FR 42179 (2025-08-29).

⛔ **eCFR returns the wrong regulation.** The printed text of 36 CFR §§ 294.10–294.18 is the **2005
State Petitions Rule**. The 2001 Roadless Rule — the version in effect — must be read from
**66 FR 3244**.

### The denominator ladder

Percentages in the press release and Draft EIS are computed against the **potentially affected
environment**, not total roadless acreage:

| Step | Acres | Source |
|---|---:|---|
| All inventoried roadless areas | 58,419,694 | this layer |
| − Idaho (9,285,370) + Colorado (4,433,322) | −13,718,692 | this layer, `STATE IN ('ID','CO')` |
| **= rule-affected** | **44,701,002** | matches "more than 44 million acres" |
| − not on NFS lands | −400,000 | DEIS Vol I p. 30, fn. 9 |
| = on National Forest System lands | 44,300,000 | |
| − designated wilderness | −1,300,000 | DEIS Vol I fn. 14 |
| − wilderness study areas | −2,800,000 | fn. 15 |
| − Wild & Scenic wild segments | −85,000 | fn. 16 |
| **= potentially affected environment (PAE)** | **40,049,537** | **DEIS Vol I Table 12** |

The West-10 share (AK, AZ, CA, MT, NV, NM, OR, UT, WA, WY) is **95.61%** of the rule-affected base
but **73.16%** of the all-IRA base — the ">95%" claim holds only on the former. **State the base with
every percentage.**

### The three headline claims

All three are press-release claims attributed to Chief Tom Schultz; the Federal Register preamble
contains none of them, and the Draft EIS supplies the methods.

| Claim | Agency method | Testable here? |
|---|---|---|
| ">40% high or very high wildfire hazard potential" | 11,479,564 ac (DEIS Vol I Table 22) ÷ PAE — **41.8% excluding Alaska**, 28.7% including it. WHP has no Alaska coverage; ">40%" is the excluding-Alaska figure. | ❌ needs WHP (pending) |
| "only 5% received hazardous fuels treatments since 2014" | FACTS, *completed*, **fiscal** years 2014–2024, ÷ PAE (DEIS Vol I p. 94). **Activity codes never disclosed**, nor whether overlapping records were dissolved or summed. | ⚠️ partially — code choice is yours to publish |
| "more than a quarter — 11.3M ac — already near existing roads" | Within **0.5 mi either side** of NFS or other authorized public roads (NRM Sept 2025) = 28.3% of PAE. The agency's own Economic Analysis gives **13.3M / 30.8%** against the 44.7M base. | ❌ needs the roads layer (pending) |

⚠️ **Montana does not reconcile.** The announcement calls Montana roadless area "nearly 60 percent of
Forest Service land," but Montana holds 6,395,401 IRA acres — 60% would require a 10,659,001-acre NFS
denominator, well below Montana's actual NFS acreage. Settling it needs the administrative-forest
layer (pending). Unresolved, not false.

⚠️ **The rule does not prohibit fuels work.** § 294.13(b)(1)(ii) of the 2001 rule already permits
cutting small-diameter timber "to maintain or restore the characteristics of ecosystem composition and
structure, such as to reduce the risk of uncharacteristic wildfire effects," and the DEIS concedes the
rule "does not prohibit and has not entirely prevented hazardous fuels reduction in IRAs" and that
"there are no prohibitions on the use of prescribed fire." The agency's argument is about frequency
and friction. The DEIS also cites Healey (2020), which reached the opposite conclusion from similar
agency data.

## Datasets not yet in this app

Ingest is tracked in [boettiger-lab/data-workflows](https://github.com/boettiger-lab/data-workflows)
under the [`roadless`](https://github.com/boettiger-lab/data-workflows/issues?q=is%3Aissue+label%3Aroadless)
label, coordinated by [#594](https://github.com/boettiger-lab/data-workflows/issues/594). Layers are
added here as each lands.

| Issue | Dataset | Why it matters |
|---|---|---|
| [#585](https://github.com/boettiger-lab/data-workflows/issues/585) | USFS administrative forest / proclaimed / surface ownership + ranger districts | The NFS denominators; settles the Montana claim |
| [#586](https://github.com/boettiger-lab/data-workflows/issues/586) | Wildfire Hazard Potential v2023, 270 m | Claim 1 |
| [#587](https://github.com/boettiger-lab/data-workflows/issues/587) | FPA-FOD wildfire occurrence 1992–2020 | Ignition history |
| [#588](https://github.com/boettiger-lab/data-workflows/issues/588) | USFS RoadCore + Census TIGER roads | Claim 3 |
| [#589](https://github.com/boettiger-lab/data-workflows/issues/589) | FR proposed rule + Draft EIS methods capture | The methods this page cites |
| [#590](https://github.com/boettiger-lab/data-workflows/issues/590) | LANDFIRE 2023/2024 — VCC/FRCC, EVT, EVC, FBFM40 | Forest mask, condition class |
| [#591](https://github.com/boettiger-lab/data-workflows/issues/591) | USFS Insect & Disease Detection Survey | Forest health |
| [#592](https://github.com/boettiger-lab/data-workflows/issues/592) | Wildfire Risk to Communities v2 | Exposure |
| [#593](https://github.com/boettiger-lab/data-workflows/issues/593) | MTBS perimeters + burn severity 1984–2024 | Severity, reburn |
| [#603](https://github.com/boettiger-lab/data-workflows/issues/603) | TWIG interagency fuel treatments + intersections | Interagency treatment record |

Already in the catalog and available to the assistant via SQL even though not on the map:
`copernicus-glo90` (slope), `usgs-wbd-hu12` (watersheds), `census-2024-*` (states, counties,
congressional districts), `epa-sab-v3-cws` (drinking-water source areas).
