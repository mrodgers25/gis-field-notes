# Spatial joins that quietly duplicate rows on shared boundaries

**Topic:** spatial joins · **Tools:** GeoPandas, Shapely

## The problem

You join clinic locations to service-area polygons to count clinics per area. The
report says 6 clinics, but your point layer has 5. Nothing errored.

The cause is a point that sits exactly on the shared edge of two adjacent polygons.
With the `intersects` predicate, that point matches both polygons, so the join
returns two rows for one clinic. Any sum or count downstream is now inflated.
This happens often with geocoded addresses that snap to parcel or district
boundaries, and with points digitized on a street that is also a boundary line.

## The explanation

- `intersects` is true if the geometries share any point, including the boundary.
- `within` is true only if the point is inside the polygon interior or on its
  boundary in a way the polygon "contains" it. For a point on the edge, `within`
  returns `False` for both polygons, so the point is dropped instead of duplicated.
- Neither is what you want. You want each point assigned to exactly one polygon,
  with a predictable rule for ties.

The fix is to join with `intersects`, then keep one match per point using a
deterministic tie-break (here, the lowest area ID). Then assert that the output row
count equals the input row count.

## Working snippet

No credentials needed. `pip install geopandas`.

```python
import geopandas as gpd
from shapely.geometry import Point, box

# Two adjacent service areas sharing the edge x = 10
areas = gpd.GeoDataFrame(
    {"area_id": ["A", "B"]},
    geometry=[box(0, 0, 10, 10), box(10, 0, 20, 10)],
    crs="EPSG:3857",
)

clinics = gpd.GeoDataFrame(
    {"clinic_id": [1, 2, 3, 4, 5]},
    geometry=[Point(2, 5), Point(8, 3), Point(10, 5), Point(15, 5), Point(18, 9)],
    crs="EPSG:3857",
)  # clinic 3 sits exactly on the shared boundary

naive = gpd.sjoin(clinics, areas, how="left", predicate="intersects")
print("naive rows:", len(naive), "(expected", len(clinics), ")")

# One row per point: sort so ties resolve to the lowest area_id, then de-duplicate
joined = (
    naive.sort_values(["clinic_id", "area_id"])
    .drop_duplicates(subset="clinic_id", keep="first")
    .drop(columns="index_right")
)

assert len(joined) == len(clinics), "join changed the row count"
print(joined[["clinic_id", "area_id"]].to_string(index=False))
print(joined.groupby("area_id").size().to_string())
```

Output:

```
naive rows: 6 (expected 5 )
 clinic_id area_id
         1       A
         2       A
         3       A
         4       B
         5       B
area_id
A    3
B    2
```

## Takeaways

- After any spatial join, compare row counts before and after. Make it an `assert`.
- Decide your tie-break rule on purpose and write it down. "Lowest ID wins" is
  arbitrary but repeatable, which matters more when reports get re-run.
- Points with no match (`area_id` is NaN under `how="left"`) are worth a separate
  check. They are often geocoding failures or points outside your service area.
