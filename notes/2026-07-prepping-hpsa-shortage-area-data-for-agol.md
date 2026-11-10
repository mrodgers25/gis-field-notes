# Prepping federal shortage-area data for ArcGIS Online

*Worked on: July 2026*

A health project needed dental Health Professional Shortage Area (HPSA) data on a web map. The federal download is a national shapefile, and it isn't ready to publish as-is. Most of the work was deciding what to keep and what to keep separate, so the layer would mean what a viewer thinks it means.

## What was tricky

HPSA data isn't one table. It ships in pieces, and mixing them is easy to do and hard to notice on a map:

- **Geographic and population-group polygons** describe areas or groups of people that are designated as shortage areas.
- **Facility-type records** (facilities that carry a designation) ship as a separate *point* file.
- **Status matters.** The file includes records that aren't currently in effect. A "Proposed for Withdrawal" record isn't the same as a Designated one, and drawing both in the same color overstates the problem.

The data is also in WGS84, with no projected coordinate system.

## How I handled it

I did the prep in a notebook so each decision was written down and repeatable:

1. **Filter to one state.** The project only covered one state, so I cut the national file down before publishing anything.
2. **Keep only Designated records.** I excluded Proposed for Withdrawal.
3. **Keep the polygons and the facility points separate.** Geographic and population-group polygons became their own layer. The facility-type point file stays a different layer, since it's a different geometry with a different meaning.

```python
import geopandas as gpd

hpsa = gpd.read_file("hpsa_dental_polygons.shp")

subset = hpsa[
    (hpsa["State"] == "XX")                      # the project's state
    & (hpsa["Status"].str.strip() == "Designated")
].copy()

print(len(hpsa), "->", len(subset))
```

(Column names here are illustrative; check the real ones in your download. Stripping whitespace before comparing is a cheap habit.)

## The CRS caveat

WGS84 is fine for display, and that's all I needed for the web map. But it's a geographic coordinate system, so units are degrees. If you calculate areas or distances, **reproject first** to a suitable projected CRS:

```python
subset_proj = subset.to_crs(epsg=5070)  # an equal-area CRS for the contiguous US
subset["area_sq_mi"] = subset_proj.area / 2.59e6
```

I didn't need area calculations for this layer, but I noted the caveat so nobody computes square miles from degrees later.

## Takeaways

- **Federal designation data comes in pieces.** Polygons and facility points are different things; publish them as separate layers.
- **Filter on status deliberately.** Decide what counts, and write it down.
- **Display CRS isn't analysis CRS.** WGS84 is fine to show; reproject before measuring.
- **Cut to your study area early.** A smaller file is faster to inspect, publish, and reason about.
