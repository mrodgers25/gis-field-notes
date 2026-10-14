# Lowercase globalid/objectid fields breaking joins

*Worked on: September 2026*

I was working on a distribution-tracking setup where each item handed out has a barcode, and a follow-up survey later references that same barcode. Scripts and dashboard data expressions joined the distribution records to the follow-up records by barcode. Everything that had worked against my usual layers suddenly misbehaved against one set of hosted layers.

## What was tricky

The layers had been created outside my usual publishing path, and their system fields came through as lowercase `globalid` and `objectid` instead of the `GlobalID` and `OBJECTID` I'd hard-coded everywhere. Nothing announced this. Lookups returned nothing or raised key errors, and joins came back empty or partial, which looked like a data problem (missing barcodes) rather than a schema problem.

My code assumed the casing in three places: the Python scripts reading feature attributes, the code that built edit payloads, and the Arcade data expressions in the dashboard that matched records across the two layers.

## How I solved it

First, I stopped hard-coding system field names. In Python I asked the layer what its fields were called and looked them up case-insensitively:

```python
def system_fields(layer):
    by_lower = {f["name"].lower(): f["name"] for f in layer.properties.fields}
    return by_lower["objectid"], by_lower.get("globalid")

oid_field, gid_field = system_fields(layer)
rows = layer.query(out_fields=f"{oid_field},{gid_field},barcode").features
```

Second, I normalized attribute keys right after querying, so everything downstream saw one consistent shape:

```python
def norm(attrs):
    return {k.lower(): v for k, v in attrs.items()}
```

Third, in the dashboard data expressions I made the join key the barcode itself, which is a field I control, and avoided depending on system field casing at all. Where I did need the system fields, I referenced the exact names those specific layers use, and left a comment saying why.

I also added a small check at the start of each script that compares the layer's actual field names against what the script expects and fails loudly if something is off. That ties in with a lesson from an earlier note: ArcGIS will often take a mismatched payload without complaint, so the check has to be mine.

## Takeaways

- **Field names are part of the contract, including their case.** Don't assume `OBJECTID` and `GlobalID` look the same on every hosted layer, especially ones that didn't come through your usual publishing workflow.
- **Read the schema, don't remember it.** Pulling names from the layer's properties costs a few lines and removes a whole class of silent failures.
- **Join on a key you own.** A barcode (or other business key) is more stable and more portable than a system-generated field.
- **Empty joins are a clue.** When a join returns nothing, check the field names before assuming the data is missing.
