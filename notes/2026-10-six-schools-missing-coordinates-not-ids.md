# Six schools that vanished: missing coordinates, not missing IDs

*Worked on: October 2026*

A statewide schools layer is rebuilt every year from source data: a base list of schools, with score tables joined on by school ID, then published as a hosted point layer. When I compared counts, the source had 8,461 schools and the published layer had 8,455. Six schools had disappeared somewhere along the way.

## The wrong first guess

The obvious suspect was the joins. If a school's ID didn't match across tables, maybe it fell out. That's how an inner join behaves, and a mismatched ID format (leading zeros, text vs. number) is a common cause.

But the joins weren't inner joins. They keep every row from the base list. A school with no matching ID in a score table doesn't vanish; it just comes through with blank score fields. So an ID mismatch would show up as empty values on the map, not as a missing point. The row count after the joins was still 8,461.

## What the six had in common

The loss happened one step later, when the table was turned into point features. I pulled the rows that weren't in the published layer and looked at them side by side. All six had no latitude or longitude. With nothing to build a geometry from, they were dropped when the table became points. No error, no warning, just six fewer features.

The check that found it is simple: compare counts at every step, and when they change, look at exactly which rows went missing.

```python
import pandas as pd

src = pd.read_csv("schools_source.csv", dtype={"school_id": str})
print(len(src))  # count before the joins

joined = src.merge(scores, on="school_id", how="left")
print(len(joined))  # still the same count: left join keeps every school

no_coords = joined[joined["latitude"].isna() | joined["longitude"].isna()]
print(len(no_coords))
print(no_coords[["school_id", "school_name"]])
```

Reading `school_id` as a string matters here too. If pandas reads it as a number, any leading zeros are lost and the joins start producing blank scores for real schools, which is a different problem that looks a lot like this one.

## The fix

The replacement annual notebook doesn't just pass through whatever coordinates the source has. Before building points, it:

1. Finds every school with a missing latitude or longitude.
2. Fills those from the state school directory, matched on school ID.
3. Flags anything still unlocated in a separate list, so a person can look at it instead of the school silently dropping out.

```python
directory = pd.read_csv("state_school_directory.csv", dtype={"school_id": str})
lookup = directory.set_index("school_id")[["latitude", "longitude"]]

missing = joined["latitude"].isna() | joined["longitude"].isna()
for col in ["latitude", "longitude"]:
    joined.loc[missing, col] = joined.loc[missing, "school_id"].map(lookup[col])

still_missing = joined[joined["latitude"].isna() | joined["longitude"].isna()]
if len(still_missing):
    still_missing.to_csv("unlocated_schools.csv", index=False)
    print(f"{len(still_missing)} schools still have no coordinates; see unlocated_schools.csv")
```

The flag is the part I care most about. Filling from the directory covers the common case, but if a school isn't in the directory either, the notebook says so out loud.

## Takeaway

When records go missing in a pipeline, check what each step actually does before blaming the usual suspect. A left join can't lose rows; converting a table to points can. Count at every stage, look at the specific rows that disappeared, and make sure any step that can drop data reports what it dropped.
