# Geocoding fallbacks that don't lie

*Worked on: August 2026*

When I loaded a donor CRM into a hosted feature layer, the geocoding step had to deal with records that didn't fit the normal street-address mold. The easy path is to let every failure fall through to some default coordinate. That default is how points end up on "Null Island," at 0,0 in the Gulf of Guinea, looking like real data.

## The situation

Most records had ordinary street addresses and geocoded fine. But a real CRM has other kinds of rows too:

- Military addresses (APO/FPO/DPO), where the "state" is AA, AE, or AP and the "city" is not a place you can map.
- Records with a ZIP code but no usable street address.
- Records with no address at all.

## What was tricky

A geocoder will often return *something* for junk input, or a caller's error handling will substitute zeros. Either way you get a point that looks valid but is wrong. A map with a cluster at 0,0 is at least obvious. A point silently placed at the wrong ZIP is worse, because it isn't obvious at all.

The military addresses were the sneaky case. AA, AE, and AP look like state codes, so they pass a naive "has a state" check, and then the geocoder either fails or matches something unrelated.

## How I fixed it

I made the fallback order explicit, and made each level honest about how precise it was:

1. Full address geocode, when the record has a real street address.
2. ZIP centroid, when there's a ZIP but no usable street address. This includes military addresses, where the ZIP is the only mappable piece.
3. Null geometry, when there's nothing to go on. The feature is written without a shape rather than with 0,0.

I also stored which method produced each location, so anyone using the layer can tell an address match from a ZIP centroid.

```python
MILITARY_STATES = {"AA", "AE", "AP"}

def locate(rec, geocode, zip_centroids):
    zip5 = (rec.get("zip") or "")[:5]
    is_military = (rec.get("state") or "").upper() in MILITARY_STATES

    if rec.get("street") and not is_military:
        hit = geocode(rec)
        if hit:
            return hit, "address"

    if zip5 in zip_centroids:
        return zip_centroids[zip5], "zip_centroid"

    return None, "none"   # null geometry, never 0,0
```

When the result is `None`, I leave the `geometry` key out of the feature entirely, so the layer stores a null shape.

## Takeaway

A fallback should get less precise, never less truthful. Decide up front what each failure mode should produce, record the precision, and prefer "no location" over a fake one. Null geometry is easy to filter and easy to fix later; a point at 0,0 gets mistaken for data.
