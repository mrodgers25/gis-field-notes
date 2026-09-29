# ArcGIS silently drops attributes with the wrong field names

*Worked on: August 2026*

I recently ran a backfill that pushed roughly 34,000 records from a donor CRM into a hosted feature layer in ArcGIS Online. The job finished with no errors. Every batch reported success. Then I opened the layer and found that most of the attributes were blank.

## What went wrong

The geometry had landed fine, and the rows existed. But the attribute columns were mostly empty. My code built each feature's attributes dictionary using CRM-style keys, things like the names the CRM export uses for its columns. The layer's real field names were different. Different casing, different naming conventions, and in some cases entirely different words.

ArcGIS doesn't complain about this. When you send `edit_features` an attribute key that isn't a field on the layer, it ignores that key. The edit still counts as a success, because the feature was added. Only the keys that happened to match (a couple of fields that were named identically on both sides) were written. Everything else disappeared without a warning.

That's the tricky part: the API's success response tells you the edit was applied. It doesn't tell you that your payload was applied as you intended.

## How I solved it

I had to re-run the backfill, so I added a validation step that runs before any write. It compares the keys in the payload against the layer's actual schema and stops the job if anything doesn't match.

```python
def validate_payload_keys(layer, features):
    layer_fields = {f["name"] for f in layer.properties.fields}
    payload_keys = set()
    for feat in features:
        payload_keys.update(feat["attributes"].keys())

    unknown = payload_keys - layer_fields
    if unknown:
        raise ValueError(
            f"Payload keys not in layer schema: {sorted(unknown)}"
        )

validate_payload_keys(layer, adds)
result = layer.edit_features(adds=adds)
```

I also added a mapping dictionary from CRM column names to layer field names, so the translation lives in one place instead of being scattered through the code. After the write, I spot-check a sample of records and count non-null values per field, so a blank column is obvious right away.

## The takeaway

A success response from `edit_features` means the features were added. It doesn't mean your attributes were written. Any time I'm writing to a layer from an external source, I now do two things:

1. **Validate before writing.** Compare payload keys to the layer schema and fail loudly on any mismatch.
2. **Verify after writing.** Check null counts on a sample, or on the whole field, before calling the job done.

Silent failures are the expensive kind. A loud error at the start of a run costs a minute. A quiet one on a 34,000-record backfill costs a full re-run and a lot of second-guessing about what else might be wrong.
