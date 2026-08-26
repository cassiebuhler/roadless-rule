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
| **LANDFIRE** | Landscape Fire and Resource Management Planning Tools (USGS EROS + USDA Forest Service) |
| **NRM** | Natural Resource Manager — the Forest Service system of record for roads |
| **TIGER** | Topologically Integrated Geographic Encoding and Referencing (US Census Bureau) |
| **NFST** | National Forest System Trails (the USFS trails inventory) |
| **GTLF** | Ground Transportation Linear Features (the BLM trails inventory) |
| **NTS** | National Trails System — the designated Scenic, Historic and Recreation trails |
| **NRI** | Nationwide Rivers Inventory (National Park Service) |

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

### National Forest System extent · USFS 2025

| Layer | Features | `SUM(GIS_ACRES)` | Collection |
|---|---:|---:|---|
| `Forest Service ownership, 193.2M ac · USFS 2025` | 97,493 parcels | 193,174,461 | `nfs-surface-ownership` |
| `Proclaimed boundary, 225.1M ac · USFS 2025` | 154 units | 225,145,181 | `proclaimed-forest` |
| `Administrative boundary, 236.8M ac · USFS 2025` | 112 units | 236,835,251 | `administrative-forest` |
| `Ranger districts, 237.1M ac · USFS 2025` | 503 districts | 237,098,674 | `ranger-district` |

Publisher: USDA Forest Service, Enterprise Data Warehouse. Coverage: national. Vintage: 2025-06-22
snapshot. License: public domain. Sources — [`S_USA.SurfaceOwnership.zip`](https://data.fs.usda.gov/geodata/edw/edw_resources/shp/S_USA.SurfaceOwnership.zip),
[`S_USA.ProclaimedForest.zip`](https://data.fs.usda.gov/geodata/edw/edw_resources/shp/S_USA.ProclaimedForest.zip),
[`S_USA.AdministrativeForest.zip`](https://data.fs.usda.gov/geodata/edw/edw_resources/shp/S_USA.AdministrativeForest.zip),
[`S_USA.RangerDistrict.zip`](https://data.fs.usda.gov/geodata/edw/edw_resources/shp/S_USA.RangerDistrict.zip).

⚠️ **Only the first layer is ownership. The other three are administrative envelopes.** They enclose
inholdings and private, state and other-federal parcels the Forest Service does not own, which is why
they run 32–44M acres above it. They are drawn as **outlines with no fill** to keep that distinction
visible on the map: ownership is land, the rest are boundaries.

Using an envelope as the base for a "share of Forest Service land" figure inflates the denominator by
up to 22.6% and deflates the resulting percentage by about 18% relative. **Surface ownership is the
right base**, and it is corroborated independently: PAD-US 4.1 `fee` filtered to `Mang_Name = 'USFS'`
gives 193,275,732 acres — 101,271 acres apart (0.05%) from two unrelated publishers — and both agree
with the announcement's "193 million."

The ownership layer is filtered in the map to `OWNERCLASS = 'USDA FOREST SERVICE'`. The full parquet
retains what that filter hides, and the assistant can query it:

| `OWNERCLASS` | Parcels | Acres |
|---|---:|---:|
| `USDA FOREST SERVICE` | 97,493 | 193,174,461 |
| `NON-FS` | 19,663 | 10,993,819 |
| `UNPARTITIONED RIPARIAN INTEREST` | 34 | 130,508 |

Those 11.0M acres of `NON-FS` parcels are the inholdings themselves — retained deliberately, since
they are what makes this layer more precise than an envelope.

⚠️ **There is no state column in any of the four.** Attributing acreage to a state requires a spatial
or hex join — `census-2024/state` is native H3 resolution 8, and these layers are native resolution 10
with h9/h8/h0 parents, so they join cell-for-cell against `roadless-areas-2001` as well. A plain
`ST_Intersects` sum is wrong: it credits a parcel's entire acreage to every state its geometry
touches, which overstates Montana by roughly 9M acres.

### Comparison strata

| Layer | Publisher | Coverage | Vintage | License | Collection |
|---|---|---|---|---|---|
| `Designated wilderness · PAD-US 4.1` | USGS Gap Analysis Project | National | 4.1 (released Mar 2025) | Public domain | `pad-us-4.1-combined` |
| `Wilderness study areas · PAD-US 4.1` | USGS Gap Analysis Project | National | 4.1 | Public domain | `pad-us-4.1-combined` |
| `Wild & Scenic Rivers, wild segments · PAD-US 4.1` | USGS Gap Analysis Project | National | 4.1 | Public domain | `pad-us-4.1-combined` |

DOI [10.5066/P96WBCHS](https://doi.org/10.5066/P96WBCHS). All three are the PAD-US **combined** layer,
distinguished by `alias` and complementary `Des_Tp` filters on one shared STAC asset.

**These three strata are the deduction behind the fourth denominator.** DEIS Vol I Table 12 nets
wilderness, wilderness study areas and Wild & Scenic River *wild* segments out of the ~44.3M NFS
acres to reach the 40,049,537-acre potentially affected environment — a deduction of **4,250,463
acres**. Each layer is drawn for all managers, not just USFS, so **no layer is itself the deduction**;
the USFS-managed subset is the part that can intersect roadless area:

| Layer | `Des_Tp` filter | All managers | USFS-managed |
|---|---|---:|---:|
| Designated wilderness | `WA` | 994 feat · 112.5M ac | 475 feat · 37.11M ac |
| Wilderness study areas | `WSA` | 722 feat · 24.16M ac | 42 feat · 3.40M ac |
| W&SR wild segments | `WSR` + `Loc_Ds` | 163 feat · 1.36M ac | 91 feat · 0.58M ac |

⚠️ **`Loc_Ds` is what makes the W&SR layer honest.** PAD-US files all Wild & Scenic River records
under `Des_Tp = 'WSR'` (910 features, 5.3M ac) regardless of classification; only `Loc_Ds`
distinguishes wild from scenic and recreational, and designated from eligible and suitable. The layer
filters to `Loc_Ds IN ('Wild', 'Designated - Wild')` because Table 12 deducts **designated** wild
segments. `Eligible - Wild` (134 feat, 0.25M ac) and `Suitable - Wild` (56 feat, 0.13M ac) are
deliberately excluded — they are candidates, not designations. Never relabel this layer as all W&SR;
that would be 4× the acreage and the wrong denominator component.

⚠️ **The combined layer contains overlapping polygons** for the same ground (separate fee, easement,
proclamation, marine and designation rows), so `SUM(GIS_Acres)` over it is meaningless at national
scale — unfiltered it totals 8.68B acres across a country with ~2.3B acres of land. Deduplicate by
unit or intersect geometries first. The three layers above can also overlap each other (a wild river
segment can run through wilderness), so do not add their acreages.

⚠️ **Four units are excluded from the wilderness layer because their geometries wrap the
antimeridian.** `Papahanaumokuakea Marine National Monument`, `Pacific Remote Islands Marine National
Monument`, `Alaska Maritime National Wildlife Refuge` and `Aleutian Islands Wilderness Area` are
stored with unwrapped longitudes — a `bbox` spanning the full −180→180 over a narrow latitude band —
so MapLibre draws each one as a horizontal stripe across the entire map. 47 features in `combined`
have this defect. The `WSA` and `WSR` strata contain **none** of them and carry no exclusion —
verified, not assumed. All four units are **FWS**-managed, so none is a comparison stratum for
National Forest System roadless area. It is nonetheless a **workaround**
([data-workflows#617](https://github.com/boettiger-lab/data-workflows/issues/617)): the fix is to
split these polygons at ±180 in the PAD-US build, and until that lands the map understates Alaska and
Pacific protected area. The parquet is unaffected and the assistant can query all of it.

⚠️ **PAD-US `fee` and `easement` are deliberately not mapped here.** `fee` filtered to
`Mang_Name = 'USFS'` is 193.3M acres in 469 polygons — essentially the whole National Forest System,
which *contains* every IRA acre rather than distinguishing any of them. `easement` is 98%
non-federal (62.3M of 63.4M ac), so it is very nearly disjoint from federal NFS roadless area.
Neither is a comparison stratum for this audit. Both remain queryable via the catalog.

⚠️ **PAD-US carries its own `Des_Tp = 'IRA'` records** (~58.2M ac, close to the USFS 58.4M). It is a
different rendering of the same inventory, not an independent one — never add it to the USFS roadless
layer.

### Trails & recreation access

| Layer | Publisher | Coverage | Vintage | License | Collection |
|---|---|---|---|---|---|
| `USFS trails, 134,983 mi · Federal Trails 2026` | USDA Forest Service (NFST) | National | 2026 | Public domain | `federal-trails-2026` |
| `NPS & BLM trails, 25,916 mi · Federal Trails 2026` | NPS + BLM (GTLF) | National | 2026 | Public domain | `federal-trails-2026` |
| `National Trails System routes, 12,488 mi · Federal Trails 2026` | USFS + NPS + BLM | National | 2026 | Public domain | `federal-trails-2026` |
| `Rivers with outstanding values, 90,476 mi · NPS NRI 2024` | National Park Service | 50 states + PR + territories | 2024 | Public domain | `american-rivers-nri-2024` |

**Why recreation data is in a rule-audit app.** The 2001 rule prohibits road construction and
reconstruction. It does not close land to the public. A map carrying only roads invites the
inference that a roadless area is unreachable; ~135,000 miles of Forest Service trail says
otherwise. These layers make that testable rather than rhetorical.

The first three layers are one dataset (`federal-trails-2026`, 127,619 segments) filtered on
`admin_agency` and `nts_designation`. `length_miles` is recomputed in EPSG:5070, not carried from
the source.

| `admin_agency` | Segments | Miles |
|---|---:|---:|
| `USFS` | 77,234 | 134,983 |
| `NPS` | 31,281 | 17,046 |
| `BLM` | 19,104 | 8,870 |

⛔ **The National Trails System layer is a subset, not a third category.** Its 4,382 segments are
already inside the USFS and NPS/BLM layers; adding the three mileages triple-counts the designated
routes. The 16 designation codes are `NRT` (1,853 segments), `PCT` (792), `NST` (700), `CDT` (404),
`AT` (225), `NCT` (203), `INHT` (64), `ANZA` (45), `PENHT` (36), `FNST` (30), `ANT` (12), `IANST`
(8), `OSNHT` (5), `PNT` (2), `NTNST` (2) and `PHT` (1).

⛔ **A trail is not a road, and this app never counts it as one.** 36 CFR 294.11 defines a road as
"a motor vehicle travelway over 50 inches wide". No trail mileage enters any road-proximity figure
here — the same rule that excludes the TIGER walkway and bridle-path classes.

⚠️ **The trail schema is only partly harmonized, despite what the collection description implies.**
`admin_agency` (3 clean values) and `nts_designation` (16 clean codes) are safe to filter on.
`trail_type`, `trail_class` and `trail_surface` are **not**: they carry each source agency's raw
vocabulary. `trail_type` has 41 distinct values that collapse to 31 once case is folded — `4WD Low`,
`4WD LOW` and `4wd Low` are three rows for one thing — plus 3,367 blank or `<Null>` records.
`trail_surface` has 62 values, `trail_class` 20 (mixing bare `1`–`6` with `Class 1: Undeveloped`).
Grouping on any of the three splits one category across several rows.

⚠️ **Use the GeoParquet, not the hex, for trail-miles questions.** Lines are hexed by circumradius
buffering, so `length_miles` repeats on every cell a buffered segment touches. `SUM(length_miles)`
on hex overcounts — the same defect as `SEG_LENGTH` on RoadCore.

⛔ **NRI is an inventory, not a protection.** Listing records that the Park Service assessed a
free-flowing segment and found at least one outstanding natural, cultural or recreational value,
making it *potentially eligible* for Wild & Scenic designation. It confers nothing. The app's
designated Wild & Scenic stratum is a **different dataset** — PAD-US `Des_Tp`, under *Comparison
strata*, where it serves as a DEIS Table 12 deduction. Never combine or compare the two as though
one updated the other.

⚠️ **`Management` on the NRI layer is free text and cannot be filtered as an agency column.** 1,827
of 4,496 segments leave it blank; the rest name individual units (`Tongass National Forest`,
`Grand Staircase-Escalante National Monument`). Attributing NRI mileage to an agency needs a spatial
join, not a `GROUP BY`. `Classifica` is likewise multi-valued (`Wild, Scenic, Recreational`) and
blank on 1,667 segments, so it is not a clean stratum either.

### Fuels & fire

| Layer | Publisher | Coverage | Vintage | License | Collection |
|---|---|---|---|---|---|
| `Completed treatments, FY2014+ · USFS FACTS 2026` | USDA Forest Service | National (NFS lands) | 2026-06 snapshot | Public domain | `facts-common-attributes-2026-06` |
| `Fire perimeters 1835–2020 · USGS 2021` | U.S. Geological Survey | National | 2021 release | Public domain | `usgs-fires-2021-combined` |
| `Wildland-urban interface · SILVIS 2020` | SILVIS Lab, UW–Madison | National (census blocks) | 2020 (v4, decades 1990–2020) | CC-BY-4.0 | `silvis-wui-2020` |
| `Wildfire hazard index · WHP 2023 (CONUS)` | USDA Forest Service (Dillon 2023, 4th ed.) | **CONUS only** | v2023, 270 m | Public domain | `whp-2023-continuous-conus` |
| `Wildfire hazard index · WHP 2023 (Alaska)` | USDA Forest Service (Dillon 2023, 4th ed.) | **Alaska only** | v2023, 270 m | Public domain | `whp-2023-continuous-ak` |
| `Wildfire perimeters 1984–2024 · MTBS` | MTBS (USGS / USFS) | National incl. Alaska | 1984–2024 | Public domain | `mtbs-perimeters-1984-2024` |
| `Prescribed fire perimeters 1984–2024 · MTBS` | MTBS (USGS / USFS) | National incl. Alaska | 1984–2024 | Public domain | `mtbs-perimeters-1984-2024` |
| `Burn severity by year · MTBS (CONUS)` | MTBS (USGS / USFS) | **CONUS only** | 39 annual years, 1984–2024 | Public domain | `mtbs-severity-1984-2024-conus` |
| `Burn severity by year · MTBS (Alaska)` | MTBS (USGS / USFS) | **Alaska only** | 36 annual years, 1984–2023 | Public domain | `mtbs-severity-1984-2024-ak` |
| `Ignitions by cause 1992–2024 · FPA-FOD` | USDA Forest Service (Short, 7th ed.) | National incl. AK, HI, PR, VI, Guam | 1992–2024 | Public domain | `fpa-fod-1992-2024` |
| `Large-fire ignitions ≥1,000 ac 1992–2024 · FPA-FOD` | USDA Forest Service (Short, 7th ed.) | National | 1992–2024 | Public domain | `fpa-fod-1992-2024` |

WHP source DOI [10.2737/RDS-2015-0047-4](https://doi.org/10.2737/RDS-2015-0047-4). The two severity
layers carry a **year selector** rather than one panel entry per year.

⚠️ **WHP is hazard, not risk.** It indexes relative potential for high-intensity fire that would be
difficult to control. The source metadata states plainly that it is **not** a measure of risk to
homes, communities or people and does not account for what is exposed. Conflating the two is the
central framing move in the announcement.

⚠️ **The mapped WHP layer is the continuous index; the classified 1–5 product is SQL-only.** The
classified rasters carry no colour table upstream, so a map legend for them would be invented rather
than published — they are available to the assistant through the hex asset, which is where the
agency-figure reproduction belongs in any case.

⛔ **CONUS and Alaska WHP are separate layers because the classification is domain-relative.** The
class breaks are percentiles computed *within each domain*, so "Very High" is index > 1,985 in CONUS
but > 8,912 in Alaska — 4.5× apart. Never pool the classified domains; use the continuous index for
any CONUS-vs-Alaska comparison. Each layer's colour ramp saturates at its own Very High break, which
is why the two use different rescale bounds. They are consequently **two separate panel toggles** —
a user who wants national wildfire hazard has to turn on both. One toggle driving both halves needs
a framework change, tracked at
[geo-agent#349](https://github.com/boettiger-lab/geo-agent/issues/349); mosaicking the two into one
national raster is **not** the fix, because a single shared stretch would saturate >10% of Alaska
into the top colour and contradict the zero-high-hazard finding below.

⚠️ **Alaska's high/very-high WHP acreage is zero, not missing** — USFS land in Alaska is Tongass and
Chugach coastal temperate rainforest, at the bottom of the *Alaska* hazard distribution. DEIS Table 22
labels the cell "Not Available"; the measured answer is 0. See the claims table below.

⚠️ **MTBS perimeters mix fire types, and the two map layers do not sum to the dataset.**
`Incid_Type` holds `Wildfire` (16,960), `Prescribed Fire` (8,870), `Unknown` (4,689) and
`Wildland Fire Use` (211) — the last documented nowhere in the source FGDC metadata, which describes
the field as `WF`/`Rx`/`UNK`. The wildfire layer covers `Wildfire` + `Wildland Fire Use`; the
prescribed-fire layer covers `Prescribed Fire`; **the 4,689 `Unknown` records appear in neither.**

⚠️ **MTBS severity has two hex assets and they are not interchangeable.** `…-hex` gives the dominant
class per cell (winner-take-all); `…-hex-fractions` gives each class's fractional coverage via a
`frac` column. Area and share questions need the fractions asset. Classes 5 (Increased Greenness) and
6 (Non-Processing Area Mask) are not severity levels; exclude them, and class 0 (Background), from any
severity denominator.

⚠️ **Severity is not a complete annual record.** Six source mosaics are permanently unavailable
upstream — CONUS **2004** and **2017**, Alaska **1987, 1995, 2001, 2013** — and Alaska has no 2024.
A missing year means *not mapped*, never "nothing burned". Perimeters cover all 41 years.

⚠️ **Reburn makes "acres burned" ambiguous.** 45,128,351 H3 cells burned at least once, against
57,552,314 (fire, cell) pairs — cumulative fire-years exceed unique ground by **27.5%**, and one cell
burned 19 times. Any total must say which it means.

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

FPA-FOD source DOI [10.2737/RDS-2013-0009.7](https://doi.org/10.2737/RDS-2013-0009.7) —
2,661,383 ignition points. Both layers colour by `NWCG_CAUSE_CLASSIFICATION`; the large-fire layer is
filtered to `FIRE_SIZE_CLASS IN ('F','G')`.

⛔ **FPA-FOD owns ignition counts; MTBS does not.** MTBS maps burned area above a size threshold and
undercounts ignitions badly, since most fires never reach it. The two join per fire on `MTBS_ID`.

⚠️ **The cause mix inside roadless areas inverts the national pattern.** Nationally: 2,034,178 human
ignitions (76.4%) vs 356,409 natural (13.4%). Inside rule-affected roadless areas: **18,166 natural
(71.6%) vs 6,697 human (26.4%)**. Nationally, natural ignitions are also likelier to become large
fires — 8,201 reached ≥1,000 acres against 6,999 human-caused. State the base with either figure.

⚠️ **Point precision has a ~1 km floor.** Many records are geolocated from PLSS descriptions, so any
distance band below about a kilometre is not defensible — including the 0.5-mile (~800 m) buffer the
roads claim uses. FPA-FOD is adjacent to claim 3 but cannot substitute for the roads layer. `FPA_ID`
is the available precision proxy.

⚠️ **`NWCG_GENERAL_CAUSE` carries no upstream domain** — its 13 values were enumerated from the
ingested data, not from documentation. The `Missing data/not specified/undetermined` bucket is large
(270,796 records) and belongs to neither cause. `OWNER_DESCR` holds both `PRIVATE` and `Private`;
compare case-insensitively.

⚠️ **This is the 7th edition, not the 6th plus four years.** It backfills previously underrepresented
states and territories, so pre-2021 counts differ from figures quoted off the 1992–2020 edition.

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

### Roads & access

| Layer | Publisher | Coverage | Vintage | License | Collection |
|---|---|---|---|---|---|
| `NFS roads open to vehicles, 263,807 mi · USFS 2025` | USDA Forest Service (NRM) | National incl. AK, PR | 2025-05-11 snapshot | Public domain | `roadcore-fs` |
| `NFS roads closed & stored (ML1), 103,945 mi · USFS 2025` | USDA Forest Service (NRM) | National incl. AK, PR | 2025-05-11 snapshot | Public domain | `roadcore-fs` |
| `All motor-vehicle roads, 16,470,232 segments · TIGER 2025` | US Census Bureau | National incl. AK, HI, PR | 2025-09-22 | Public domain | `census-2025/roads` |
| `Highways & secondary roads, 268,817 segments · TIGER 2025` | US Census Bureau | National incl. AK, HI, PR | 2025-09-22 | Public domain | `census-2025/roads` |
| `Walkways & paths, 20,667 segments — NOT roads under 36 CFR 294.11 · TIGER 2025` | US Census Bureau | National incl. AK, HI, PR | 2025-09-22 | Public domain | `census-2025/roads` |

The first two layers are one dataset (`roadcore-fs`, 367,666 segments, 368,103 official miles)
filtered on `OPER_MAINT_LEVEL`. The split is not cosmetic — it is the point of the layer.

| `OPER_MAINT_LEVEL` | Segments | Official miles | Share |
|---|---:|---:|---:|
| `2 - HIGH CLEARANCE VEHICLES` | 183,849 | 199,311 | 54.1% |
| **`1 - BASIC CUSTODIAL CARE (CLOSED)`** | **148,694** | **103,945** | **28.2%** |
| `3 - SUITABLE FOR PASSENGER CARS` | 24,631 | 49,919 | 13.6% |
| `4 - MODERATE DEGREE OF USER COMFORT` | 6,825 | 11,351 | 3.1% |
| `5 - HIGH DEGREE OF USER COMFORT` | 3,312 | 3,226 | 0.9% |
| null / `NA` / `0 - NOT MAINTAINED` | 355 | 352 | 0.1% |

⛔ **Level 1 roads are closed to motor vehicles and held in storage** — often impassable, frequently
revegetated. They are 28.2% of the system. Any "this land is near an existing road" figure changes
substantially depending on whether they count, which is why they are drawn as a separate, dashed
layer rather than folded into one road line. These three totals reproduce the Forest Service's own
published figures: ~368,000 mi system, ~65,000 mi (18%) passenger-car, ~200,000 mi (54%)
high-clearance.

The last three are one dataset (`census-2025/roads`, 16,490,899 features across 3,233 counties and
county equivalents) filtered on `MTFCC`. TIGER carries no length column, so these layers are counted
in segments, not miles — a TIGER segment and a RoadCore segment are not the same unit and the two
counts are not comparable.

| `MTFCC` | Meaning | Features | In which layer |
|---|---|---:|---|
| `S1400` | Local neighborhood road, rural road, city street | 14,009,547 | motor-vehicle |
| `S1740` | Private road for service vehicles (logging, oil field, ranch) | 1,240,392 | motor-vehicle ⚠️ contestable |
| `S1500` | Vehicular trail (4WD) | 307,839 | motor-vehicle |
| `S1200` | Secondary road | 251,196 | motor-vehicle **and** highways |
| `S1630` | Ramp | 199,684 | motor-vehicle |
| `S1750` | Internal US Census Bureau use | 196,158 | motor-vehicle ⚠️ contestable |
| `S1640` | Service drive along a limited-access highway | 137,698 | motor-vehicle |
| `S1730` | Alley | 70,210 | motor-vehicle |
| `S1780` | Parking lot road | 39,481 | motor-vehicle |
| `S1100` | Primary road | 17,621 | motor-vehicle **and** highways |
| `S1710` | Walkway / pedestrian trail | 16,087 | **excluded** — paths layer |
| `S1820` | Bike path or trail | 4,323 | **excluded** — paths layer |
| `S1810` | Winter trail | 406 | motor-vehicle ⚠️ marginal |
| `S1720` | Stairway | 234 | **excluded** — paths layer |
| `S1830` | Bridle path | 23 | **excluded** — paths layer |

⚠️ **RoadCore is Forest Service roads only.** `SYSTEM` is `NFSR` and `JURISDICTION` is `FS` on every
record — state, county and private roads are absent by construction. The DEIS buffered "National
Forest System roads *and* other authorized public roads" (Vol I fn. 10); TIGER supplies that second
half. A proximity analysis on either source alone undercounts.

⛔ **The two sources overlap and have not been deduplicated by anyone.** Forest Service roads
routinely reappear in TIGER as `S1500` vehicular trail or `S1740` private service road. Spot-checked
on the Bitterroot National Forest (a 0.3° × 0.3° window holding 112 RoadCore and 1,607 TIGER
segments), **97 of the 112 RoadCore segments — 87% — lie within 25 m of a TIGER line.** Never add
the two segment counts, and never sum their mileage. Combine them only as *distance to the nearest
road across both sources*.

⚠️ **The four excluded classes are excluded by regulation, not by taste.** 36 CFR 294.11 defines a
road as "a motor vehicle travelway over 50 inches wide"; `S1710` walkway, `S1720` stairway, `S1820`
bike path and `S1830` bridle path (20,667 features) are none of those. They are drawn as their own
dashed layer so the exclusion is visible rather than silent.

⚠️ **Two classes inside the mapped motor-vehicle layer are genuinely contestable, and the agency
never published its choice.** `S1740` private service roads and `S1750` internal Census use carry
**1,436,550 features between them** — 8.7% of the layer. `S1810` winter trail (406 features) is a
third, smaller judgment call. Including or excluding them moves any proximity figure, so a
reproduction that does not state which way it went is incomplete.

⚠️ **41.8% of TIGER road segments are unnamed** — 6,890,655 of 16,490,899 carry no `FULLNAME`. This
is normal for TIGER (unnamed rural roads and driveways) and does not affect geometry or proximity
work, but it means `FULLNAME` cannot be used to identify or deduplicate a road.

⚠️ **8,204 RoadCore records carry no geometry**, holding 4,745 official miles — an upstream
linear-referencing failure flagged by the source's own `LOC_ERROR` (7,430 are `ROUTE NOT FOUND`).
They are in the parquet with attributes intact but cannot appear in the tiles or hex, and cannot be
buffered by anyone, including the agency.

⛔ **No layer can reproduce the regulatory definition.** 36 CFR 294.11 counts unclassified and
temporary roads, and the Forest Service maintains no national temporary-roads database (DEIS Vol I
fn. 20). Every road-proximity figure computed here is a floor, not a match.

### Vegetation condition · LANDFIRE 2024 (in *Fuels & fire*)

| Layer | Publisher | Coverage | Vintage | License | Collection |
|---|---|---|---|---|---|
| `Vegetation condition class · LANDFIRE 2024 (CONUS only)` | LANDFIRE (USGS EROS / USFS) | **CONUS only** | LF 2024 (2.5.0), 30 m | Public domain | `landfire-2024-vcc` |

Vegetation Condition Class measures how far current vegetation has departed from its estimated
historical reference condition, on a six-step ordinal scale from Class I.A (0–16% departure) to
Class III.B (84–100%). It is the LANDFIRE product that speaks to claims about stands being overgrown
or out of their natural condition.

⛔ **Five codes are not rated for departure and must leave the denominator:** `111` water, `112`
snow/ice, `120` developed, `132` barren/sparse, `180` agriculture. They are not low-departure land —
they are unrated. Inside rule-affected roadless area they are 4.1% of res-10 cells.

Measured across rule-affected roadless area (res-10 cells, rated classes only):

| Class | Departure | Share of rated cells |
|---|---|---:|
| I.A | 0–16% | 0.9% |
| I.B | 17–33% | 28.9% |
| II.A | 34–50% | 37.2% |
| II.B | 51–66% | 18.4% |
| III.A | 67–83% | 13.9% |
| III.B | 84–100% | 0.7% |

So **14.6% of rated rule-affected roadless area is Class III** (high departure) and 33.0% is Class
II.B or worse. Report this as a share of *cells*, not acres — the per-class fractional-coverage table
that acre accounting needs is not published for this layer, and the hex carries only the dominant
class per cell.

⛔ **LANDFIRE 2024 and WHP 2023 are not the same vintage.** WHP 2023 is built from **LANDFIRE 2020
v2.2.0** fuels — three updates older. Never present them as one snapshot.

⚠️ **CONUS only** — the same Alaska hole as NLCD, hiding 14,778,681 roadless acres.

⚠️ **Three fill codes still appear in the legend, and two of them paint.** `-9999` matches the band
nodata and renders transparent, but `-1111` (Fill-Not Mapped) paints grey and `32767` (band sentinel)
paints white. This cannot be fixed from `layers-input.json`: for a raster asset the app takes both the
legend and the TiTiler colormap from the STAC `classification:classes`, and — unlike the vector path —
ignores any `legend_classes` override. Tracked upstream at
[data-workflows#628](https://github.com/boettiger-lab/data-workflows/issues/628).

## Reproducing the agency's numbers

The rulemaking under audit: **91 FR 53827** (published 2026-08-20), RIN **0596-AD66**, docket
**FS-2025-0001**, action *remove and reserve 36 CFR part 294, subpart B*. **Comments close
2026-09-21.** Notice of intent: 90 FR 42179 (2025-08-29).

Every DEIS table and page number cited below is traceable to a fixed copy: the Federal Register
notice, Draft EIS Volume I and Economic Analysis were captured on the publication date under
`catalog/usfs/roadless-rule-2026/` in `data-workflows` — tracked by
[#589](https://github.com/boettiger-lab/data-workflows/issues/589), still on branch
`worktree-roadless-rule-589` and **not yet merged to `main`**, so there is no permanent link to it
yet. `sources.tsv` and `fetch-sources.sh` record where each came from. Note that
`docs/pi-2026-16965.pdf` is the **public-inspection** version, captured before it rotated out of
circulation — that copy is the only one.

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

**A fifth base sits outside this ladder.** All four steps above are bases of *roadless* acreage. A
claim of the form "X percent of **Forest Service land**" is measured against Forest Service surface
ownership — **193,174,461 acres** nationally — which is a different quantity entirely and must never
be substituted into the ladder. See *National Forest System extent* above for why the ownership layer,
not one of the three administrative envelopes, is the correct base.

### The three headline claims

All three are press-release claims attributed to Chief Tom Schultz; the Federal Register preamble
contains none of them, and the Draft EIS supplies the methods.

| Claim | Agency method | Testable here? |
|---|---|---|
| ">40% high or very high wildfire hazard potential" | 11,479,564 ac (DEIS Vol I Table 22) ÷ PAE — **41.8% excluding Alaska**, 28.7% including it. Non-burnable and water are **in** the denominator; total acres, not forested. | ✅ **WHP 2023 is now in the app.** Compute from the hex, not COG pixels; keep CONUS and Alaska classified domains separate. Alaska's high/very-high acreage is **0**, so Table 22 is internally consistent — the flaw is labelling that cell "Not Available" rather than "0", which hides that dropping Alaska is what lifts 28.7% to 41.8%. |
| "only 5% received hazardous fuels treatments since 2014" | FACTS, *completed*, **fiscal** years 2014–2024, ÷ PAE (DEIS Vol I p. 94). **Activity codes never disclosed**, nor whether overlapping records were dissolved or summed. | ⚠️ partially — code choice is yours to publish |
| "more than a quarter — 11.3M ac — already near existing roads" | Within **0.5 mi either side** of NFS or other authorized public roads (NRM Sept 2025) = 28.3% of PAE. The agency's own Economic Analysis gives **13.3M / 30.8%** against the 44.7M base. | ⚠️ **Testable in SQL, not on the map.** Both road layers are in the catalog, but only the NFS half is mapped — TIGER is SQL-only. Break results out by maintenance level: ML1 roads are closed and stored, 28.2% of the system. No layer can reproduce 36 CFR 294.11, which counts temporary roads the agency does not inventory. |

⚠️ **Montana does not reconcile — and the NFS extent layers now show why.** The announcement calls
Montana roadless area "nearly 60 percent of Forest Service land." Montana holds 6,395,401 IRA acres,
so 60% would require an NFS base of about 10,659,001 acres. No Forest Service extent layer is that
small: against Montana surface ownership the share is roughly **37%**, and against the proclaimed
boundary it is lower still. The claim is therefore not a denominator ambiguity — **no available base
produces 60%**. The announcement publishes no method, so the accurate statement is that the figure
cannot be reproduced, not that it is false.

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
| [#590](https://github.com/boettiger-lab/data-workflows/issues/590) | LANDFIRE 2024 — **EVC** (canopy cover) and **FBFM40** (fuel models) only | VCC, the departure layer, is now in the app |
| [#591](https://github.com/boettiger-lab/data-workflows/issues/591) | USFS Insect & Disease Detection Survey | Forest health |
| [#592](https://github.com/boettiger-lab/data-workflows/issues/592) | Wildfire Risk to Communities v2 | Exposure |
| [#603](https://github.com/boettiger-lab/data-workflows/issues/603) | TWIG interagency fuel treatments + intersections | Interagency treatment record |
| [#609](https://github.com/boettiger-lab/data-workflows/issues/609) | Mesic Analysis Platform — 30 m mesic persistence, valley bottoms | Water-resource condition |
| [#610](https://github.com/boettiger-lab/data-workflows/issues/610) | USGS INHABIT v4 — fire-promoting invasive plants | Invasion-driven fire risk |
| [#611](https://github.com/boettiger-lab/data-workflows/issues/611) | Wildfire Risk to Communities v2 — populated areas | Exposure, populated-area basis |

Already in the catalog and available to the assistant via SQL even though not on the map:
`census-2025/roads` (TIGER roads — see below), `copernicus-glo90` (slope), `usgs-wbd-hu12`
(watersheds), `census-2024-*` (states, counties, congressional districts), `epa-sab-v3-cws`
(drinking-water source areas).

Two datasets here are SQL-only, for different reasons — both worth recording.

⚠️ **`census-2025/roads`.** Its STAC collection declares a
PMTiles asset at `census-2025/roads.pmtiles`, but that file **does not exist** — the build published
the GeoParquet and the H3 hex without it. A declared asset is not a live asset; the map layer was
left out rather than pointing at a 404.
