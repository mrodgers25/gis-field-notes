# Reuse coordinates in a migration with an address hash

*Worked on: August 2026*

During a migration from an old hosted layer to a new one, every record needed coordinates again. The obvious path was to run the whole set back through the geocoder. But most of those addresses had already been geocoded once, and the old layer was sitting right there holding the answers.

## The situation

The new layer was fed from a donor CRM, and the old layer already held geometry for most of the same people. Re-geocoding everything would have cost credits and a lot of runtime for results I mostly already had.

## What was tricky

There was no shared ID I could trust between the old layer and the new load, so I couldn't just join on a key. The address itself was the only thing the two had in common, and raw address strings don't match reliably. `123 Main St.` and `123 MAIN STREET` are the same place and different strings. Trailing spaces, punctuation, and casing differences all break an exact comparison.

## How I solved it

I normalized each address into a canonical form, then hashed it. Same normalized address, same hash, regardless of how it was originally typed. I built a lookup from the old layer (hash to coordinates), then checked each incoming record against it.

```python
import hashlib
import re

def normalize(street, city, state, zip5):
    parts = [street, city, state, zip5]
    s = " ".join(p or "" for p in parts).upper()
    s = re.sub(r"[^\w\s]", "", s)       # drop punctuation
    s = re.sub(r"\bSTREET\b", "ST", s)  # a few common abbreviations
    s = re.sub(r"\bAVENUE\b", "AVE", s)
    return re.sub(r"\s+", " ", s).strip()

def addr_hash(*parts):
    return hashlib.sha1(normalize(*parts).encode()).hexdigest()

# old layer: hash -> existing geometry
known = {addr_hash(*row_addr(f)): f.geometry for f in old_features}

to_geocode = []
for rec in incoming:
    geom = known.get(addr_hash(rec["street"], rec["city"],
                               rec["state"], rec["zip"]))
    if geom:
        rec["geometry"] = geom      # reuse, no geocoder call
    else:
        to_geocode.append(rec)      # only these hit the geocoder
```

Only the records that missed went to the geocoder.

## The result

About 90% of records matched and needed no re-geocoding. That saved geocoding credits and cut the migration time considerably.

## The takeaway

- Before re-geocoding in a migration, check whether the answers already exist somewhere.
- Hash a normalized address rather than comparing raw strings. The normalization step is where the matching quality comes from, so keep it deterministic and apply the exact same function to both sides.
- Keep the geocoder as the fallback for true misses, not the default for everything.
