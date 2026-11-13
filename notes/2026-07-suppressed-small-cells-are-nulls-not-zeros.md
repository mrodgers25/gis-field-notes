# Suppressed small cells are nulls, not zeros

*Worked on: July 2026*

A health project needed county-level dental utilization on a map. The source was a state Medicaid dataset broken out by county and age group: one row per county per age band, with a measure name and a rate. It looked tidy. Two things in it would have quietly made the map wrong.

## What was tricky

**Blank rates.** 585 of the 3,248 county-by-age rows had no rate at all. That isn't missing data in the "someone forgot" sense. When counts are small, the publisher suppresses them so individuals can't be identified. A blank means "there's a number here, but we're not allowed to show it."

The easy mistake is to let a blank turn into 0. Some tools do it for you: a fill-na step, a pivot with a default value, or a renderer that treats null as the bottom class. On a choropleth, a 0 says "nobody in this county got dental care." That's a much stronger claim than "too few people to report." And since suppression happens where counts are small, the error would cluster in small-population counties.

**Measure text with trailing whitespace.** The measure column had variants of the same label, some with trailing spaces. They look identical in a spreadsheet, but to a pivot they're different values, so one measure gets split into two columns, each half empty.

## How I handled it

I cleaned in a notebook before anything went to ArcGIS Online:

1. **Trim the measure text first**, then check the distinct values again, so the pivot sees one label per measure.
2. **Keep suppressed rates as null** and add an explicit flag, so the map can label them instead of guessing.
3. **Pivot without a fill value.**

```python
import pandas as pd

df = pd.read_csv("dental_utilization.csv")

# Same label, different whitespace = different pivot columns
df["measure"] = df["measure"].str.strip()
print(df["measure"].unique())

# Blank means suppressed, not zero
df["rate"] = pd.to_numeric(df["rate"], errors="coerce")
df["suppressed"] = df["rate"].isna()
print(df["suppressed"].sum(), "suppressed of", len(df))

wide = df.pivot_table(
    index=["county", "age_group"],
    columns="measure",
    values="rate",
    aggfunc="first",   # no fill_value: keep nulls as nulls
).reset_index()
```

(Column names are illustrative.)

On the map side, I gave suppressed features their own symbol class and legend entry, "Suppressed (small count)," rather than letting them fall into the lowest range. A popup expression in Arcade can say the same thing:

```
IIf(IsEmpty($feature.rate),
    "Suppressed for privacy (small count)",
    Text($feature.rate, "#.0") + "%")
```

## Takeaways

- **A blank in public health data usually means suppressed.** Read the data notes, and treat it as "unknown," never 0.
- **Watch for silent fills.** Check pivots, joins, and renderers for any step that turns null into 0.
- **Give suppression its own legend entry.** Viewers should see the difference between "low" and "not reported."
- **Trim text before you pivot or group.** Invisible whitespace splits one category into several.
