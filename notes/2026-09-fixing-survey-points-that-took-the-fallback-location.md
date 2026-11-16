# Fixing 44 survey points that took the fallback location

*Worked on: September 2026*

A follow-up survey I support had a default GPS fallback location built in, so the form could still be submitted if the device never captured a fix. That safety net backfired: 44 records came in sitting on the fallback point because field workers didn't tap "Locate Me" before submitting. On the map they all stacked up in one spot, and any analysis by location was wrong for those records.

## What was tricky

The good news was that most of these records could be repaired from data we already had. Each follow-up references a barcode from an earlier distribution record, and the distribution layer had good coordinates. Of the 44, 40 had a matching distribution record by barcode. The other 4 had no match and needed manual fixes.

Two snags made the matching harder than it looked:

- **GlobalID formats didn't match.** The CSV I exported to review the bad records had GlobalIDs in uppercase with curly braces, while the live layer stored them lowercase without braces. A straight comparison matched nothing.
- **Barcodes weren't unique.** The distribution layer had 1,247 duplicate barcodes, so "look up the barcode" could return more than one candidate location.

And because this was a write to a live layer, I wanted to be very sure I touched only what I meant to.

## How I solved it

I did the fix in a notebook. First, a small function to put every GlobalID into one shape before comparing:

```python
def normalize_guid(g):
    return str(g).strip().strip("{}").lower() if g else None
```

For the duplicate barcodes, I kept the most recent distribution record by date for each barcode, so every barcode resolved to exactly one location:

```python
dist = dist_df.sort_values("date", ascending=False)
latest = dist.drop_duplicates(subset="barcode", keep="first").set_index("barcode")
```

Then I built the edits so they changed only Latitude, Longitude and the geometry, nothing else on the record:

```python
updates = []
for _, row in fallback_df.iterrows():
    if row["barcode"] not in latest.index:
        continue  # the no-match records get fixed by hand
    src = latest.loc[row["barcode"]]
    updates.append({
        "attributes": {
            "GlobalID": live_gid[normalize_guid(row["GlobalID"])],
            "Latitude": src["Latitude"],
            "Longitude": src["Longitude"],
        },
        "geometry": {"x": src["Longitude"], "y": src["Latitude"],
                     "spatialReference": {"wkid": 4326}},
    })
```

The write cell sat behind a confirmation flag, so running the whole notebook top to bottom would only build and preview the changes:

```python
CONFIRM = False  # flip to True only after reviewing `updates`

if CONFIRM:
    result = followup_layer.edit_features(updates=updates, use_global_ids=True)
else:
    print(f"Dry run: {len(updates)} updates ready, nothing written.")
```

That handled the 40 matched records. The 4 without a distribution match got fixed manually.

## Takeaways

- **A fallback location is still a location.** If a form can be submitted with a default point, some records will be, and they'll look valid until you map them.
- **Normalize IDs before you compare them.** The same GlobalID can show up in different case and with or without braces depending on where it came from.
- **Check your join key for uniqueness first.** Duplicate barcodes needed a rule (most recent by date) before any lookup could be trusted.
- **Limit the blast radius.** Update only the fields you mean to change, and put live writes behind a deliberate CONFIRM step.
