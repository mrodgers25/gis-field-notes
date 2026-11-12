# Check field names and join keys before you map

*Worked on: July 2026*

A health-access mapping project pulled together three federal datasets: a CSV of health-center sites, shortage-area data, and the CDC/ATSDR Social Vulnerability Index (SVI). Each one looked usable on its own. The trouble was in the details I could have skipped: which field means what, and which key connects one table to another.

## What was tricky

**Two fields with nearly the same name.** The health-center sites CSV had two columns that both read like "Health Center Type." They held different things. One said whether a site was a Federally Qualified Health Center or a look-alike. The other held administrative categories. If I had grabbed the first one that matched a search for "type," the symbology would have been confidently wrong, and nobody looking at the map would have known.

**Two datasets at different geographic levels, with different keys.** The shortage-area data identified places by county and state *names*. The SVI data was at the census tract level and keyed on tract GEOIDs. Those don't join directly. A name is not a GEOID, and a county is not a tract.

**A table that wasn't a layer.** The SVI download I used was attributes only, with no geometry. It needed a geographic layer to land on before it could be mapped at all.

## How I handled it

First, I stopped and profiled the fields before building anything. For each candidate column I looked at the distinct values, not just the header:

```python
import pandas as pd

sites = pd.read_csv("health_center_sites.csv", dtype=str)

type_cols = [c for c in sites.columns if "type" in c.lower()]
for col in type_cols:
    print(col)
    print(sites[col].value_counts(dropna=False).head(10), "\n")
```

Seeing the values side by side made it obvious which column was the FQHC vs. look-alike flag and which was the admin category. I picked the first one for the map and noted why.

Second, I matched each dataset to the right join key instead of forcing them together. The SVI table joined to Census TIGER tract boundaries on GEOID. I read GEOID as text on both sides so leading zeros survived:

```python
import geopandas as gpd

tracts = gpd.read_file("tiger_tracts.shp", dtype={"GEOID": str})
svi = pd.read_csv("svi_tracts.csv", dtype={"FIPS": str})

svi_tracts = tracts.merge(svi, left_on="GEOID", right_on="FIPS", how="left")
print("tracts with no SVI match:", svi_tracts["FIPS"].isna().sum())
```

The shortage-area data stayed on its own terms, keyed by county and state names, and I didn't try to force it onto the tract-based SVI layer.

## Takeaways

- **Read the values, not the headers.** Similar field names can hold very different things. A quick `value_counts()` on every candidate column saves you from a map that looks right and isn't.
- **Know each dataset's geography and key before you plan a join.** Names vs. GEOIDs, and counties vs. tracts, are different problems. Figure out which one you have first.
- **Attribute-only tables need a geometry partner.** SVI as a table goes with TIGER tracts on GEOID. Keep GEOIDs as strings so the join doesn't silently drop rows.
- **Check for unmatched rows after every join.** A count of non-matches tells you right away whether the key is doing its job.
