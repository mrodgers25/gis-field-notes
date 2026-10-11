# Protecting donor privacy for outside analysts

*Worked on: August 2026*

A nonprofit client wanted an outside analytics team to build visualizations from their donor map. The team needed to see where giving was concentrated. They did not need to see where any individual donor lives.

## The situation

The main hosted layer held donor points geocoded from home addresses, plus attributes from the donor CRM. That layer was the right source for the client's own staff, but sharing it with an external team meant sharing street-level locations. Sharing a spreadsheet export instead would have gone stale right away and moved the data out of ArcGIS Online entirely.

## What was tricky

Hiding fields isn't enough. A view layer can hide attributes like name and street address, but the geometry still exposes exactly where each point is. Anyone who can query the view can read the coordinates, so the privacy problem was in the shape, not the table.

The other constraint was freshness. The analysts needed current data, and I didn't want a manual export step that someone had to remember to run.

## How I solved it

I split it into two layers:

1. The private source layer with exact points, shared only with the client's staff.
2. A separate layer for the analysts, with every point moved to the centroid of its ZIP code and the sensitive fields left out. It's exposed to them through a view layer, so the sharing settings stay simple.

A weekly job rebuilds the analyst layer. For each donor point it looks up the ZIP, swaps the geometry for that ZIP's centroid, and writes only the fields the analysts need.

```python
def to_zip_centroid(feature, centroids):
    zip5 = (feature.attributes.get("zip") or "")[:5]
    centroid = centroids.get(zip5)
    if centroid is None:
        # No usable ZIP: publish no geometry rather than a guess
        return None
    return {
        "geometry": {"x": centroid[0], "y": centroid[1],
                     "spatialReference": {"wkid": 4326}},
        "attributes": {k: feature.attributes.get(k) for k in SAFE_FIELDS},
    }
```

`SAFE_FIELDS` is an explicit allowlist. A new field added to the source layer later doesn't flow to the outside team unless I add it on purpose.

Records without a usable ZIP are skipped instead of being given a fallback location, so nothing is placed on the map that I can't stand behind.

## The takeaway

- A view layer that hides fields doesn't protect location. If the geometry is sensitive, change the geometry.
- Use an allowlist of fields for external audiences, not a blocklist. Blocklists fail quietly when the schema grows.
- Generalizing to ZIP centroids still shows regional patterns, which is what the analysts needed, while individual homes stay out of the data they can reach.
- Automate the refresh. A privacy copy that depends on someone remembering to rebuild it either goes stale or gets bypassed.
