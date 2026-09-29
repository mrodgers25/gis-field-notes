# Never cache OBJECTIDs

*Worked on: August 2026*

While building a sync between a donor CRM and a hosted feature layer in ArcGIS Online, I took a shortcut that I later had to undo: I saved each record's OBJECTID after the first load so that later updates could go straight to `edit_features` without a lookup.

## The situation

The first load went fine. I had a mapping of CRM record to OBJECTID, and updates used it to target the right feature. It was fast and it worked, so I moved on.

## What was tricky

OBJECTID looks like an identifier, but it isn't one you own. It's assigned by the service, and republishing a layer can reassign it. Once that happens, every cached OBJECTID points at the wrong feature, or at nothing. Nothing errors loudly: an update applied to the wrong OBJECTID just edits a different record. It's the same silent-failure family as any write that "succeeds" against the wrong target.

The cache also made the pipeline fragile in a quieter way. It carried an assumption that the layer would never be rebuilt, and layers do get rebuilt.

## How I fixed it

I stopped storing OBJECTIDs anywhere. Instead:

1. Every feature carries a stable external key, the CRM record ID, in its own field on the layer. That field is indexed.
2. At write time, I query the layer for the OBJECTID that matches the key.
3. Then I build the update using the OBJECTID I just looked up.

```python
def objectids_for(layer, crm_ids):
    """Look up current OBJECTIDs by stable CRM key, at write time."""
    ids = ", ".join(f"'{i}'" for i in crm_ids)
    result = layer.query(
        where=f"crm_id IN ({ids})",
        out_fields="crm_id",
        return_geometry=False,
    )
    return {f.attributes["crm_id"]: f.attributes["OBJECTID"]
            for f in result.features}

lookup = objectids_for(layer, batch_crm_ids)

updates = []
for rec in batch:
    oid = lookup.get(rec["crm_id"])
    if oid is None:
        continue  # not in the layer yet; handle as an add
    updates.append({"attributes": {"OBJECTID": oid, **rec["attrs"]}})

layer.edit_features(updates=updates)
```

Records with no match fall out naturally as adds instead of being forced onto a stale ID. For large batches, I chunk the `IN` list so the query stays a reasonable size.

## The takeaway

- Treat OBJECTID as a temporary handle, not a key. Use it within a single operation, then forget it.
- Store your own stable identifier on the layer, from the source system, and look up the OBJECTID at write time.
- The extra query per batch is cheap compared to the cost of quietly editing the wrong records after a republish.
