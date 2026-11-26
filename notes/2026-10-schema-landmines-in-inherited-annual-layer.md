# Schema landmines in an inherited annual-update layer

*Worked on: October 2026*

I inherited a schools layer that gets refreshed once a year. Before writing a new notebook to rebuild it, I went through the existing schema field by field. Most of what I found wasn't wrong in an obvious way. Each problem had been quietly costing data or effort for a while.

## What the schema review turned up

**A 12-digit ID in an Integer field.** The school ID is 12 digits long, which is too big for a standard 32-bit Integer field. Every value had been dropped. The field existed, it just held nothing useful. IDs belong in a text field, even when they're all digits.

**County FIPS codes that lost their leading zero.** The county FIPS was stored as text, but only 4 characters wide. County FIPS codes are 5 digits, so any code starting with zero had already been converted to a number somewhere upstream and come through without the zero. Joining that to any other county data would fail for those counties.

**Field names that started with a digit.** ArcGIS doesn't allow a field name to start with a number, so it had renamed those columns with an "F" prefix. That's easy to miss until a script or an Arcade expression looks for the original name and doesn't find it.

**An alias and a name that disagreed.** One field's alias said one year while its name said the previous year. Someone had updated the label for the new year without changing the field underneath, so anyone reading the alias would trust the wrong year.

## Year-neutral field names

That last one points at the real design problem: putting the year in the field name. Every annual update meant new fields, and the dashboard had to be repointed to them.

The new notebook uses year-neutral names: `current` and `prior` instead of a specific year. The dashboard points at the same fields every year and never needs to be touched.

Casting types explicitly before publishing covers the first two problems:

```python
import pandas as pd

df = pd.read_csv(
    "schools_source.csv",
    dtype={"school_id": str, "county_fips": str},
)
df["county_fips"] = df["county_fips"].str.zfill(5)

# Year-neutral names, no leading digits
df = df.rename(columns={
    "score_this_year": "score_current",
    "score_last_year": "score_prior",
})
```

## Two overwrite rules worth knowing

Overwriting the hosted layer each year has its own catches:

1. **The CSV has to have the same filename as the original source.** Overwrite expects the file it was published from. A new file with this year's date in the name won't do it, even with identical columns. I keep the source filename fixed every year.
2. **You can't overwrite a view directly.** Only the source layer it's built on can be overwritten, and the view picks up the new data from there. If a dashboard reads from a view, find the source layer behind it and overwrite that.

```python
from arcgis.gis import GIS
from arcgis.features import FeatureLayerCollection

gis = GIS("home")
source_item = gis.content.get("<source layer item ID>")  # not the view
flc = FeatureLayerCollection.fromitem(source_item)
flc.manager.overwrite("schools_source.csv")  # same filename as the original
```

## Takeaway

When you inherit a layer that gets updated on a schedule, read the schema before you touch the data. Check field types against the actual values, especially IDs and codes with leading zeros. Look at names against aliases too. And keep years out of field names, so the yearly update is just a data refresh and not a schema change that forces you to repoint the dashboard.
