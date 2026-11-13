# Batch geocoding that put Texas donors in California

*Worked on: September 2026*

One of my pipelines for a donor CRM includes a weekly Lambda that re-geocodes addresses. It gathers the records that need coordinates, sends them to the geocoder as one batch, and writes the results back, so donors show up in the right place on the map.

## What went wrong

The code paired each input address with its result using Python's `zip()`. That meant it trusted the response to come back in the same order as the request, with the same number of items. Most of the time that held. When it didn't, every pair after the first mismatch was off.

The result was 16 legitimate Texas addresses saved with California coordinates. They weren't marked as failures, either. They were marked as confidently "geocoded," so nothing downstream had any reason to look at them again. A bad match with a good-looking status is the worst kind of failure because it hides.

The buggy pattern looked roughly like this:

```python
# Don't do this: assumes results come back in input order, one per input
for record, result in zip(records, results):
    record["x"], record["y"] = result["location"]["x"], result["location"]["y"]
    record["geocode_status"] = "geocoded"
```

`zip()` also stops quietly at the shorter list, so a missing result doesn't raise an error. It just slides everything over.

## How I fixed it

I made two changes.

**1. Check the counts before writing anything.** If the number of results doesn't match the number of inputs, the code no longer tries to work out what went where. It flags the whole batch for review and writes nothing.

**2. Match on ResultID, not position.** Each address in the batch request carries its own ID, and the geocoder returns that ID in each result's `ResultID` attribute. I now key the results by that ID and look up each record directly:

```python
def apply_batch_results(records, results):
    if len(results) != len(records):
        flag_batch_for_review(records, reason="result count mismatch")
        return

    by_id = {r["attributes"]["ResultID"]: r for r in results}

    for rec in records:
        match = by_id.get(rec["batch_id"])
        if match is None:
            flag_for_review(rec, reason="no result for this ID")
            continue
        rec["x"] = match["location"]["x"]
        rec["y"] = match["location"]["y"]
        rec["geocode_status"] = "geocoded"
```

That left the records that were already wrong. For those, I set the geocode status to a manual re-geocode sentinel value, which the weekly job treats as "needs placing." On its next run, the Lambda picked them up and geocoded them again, this time with the ID-based matching.

```python
for rec in misplaced_records:
    rec["geocode_status"] = REGEOCODE_SENTINEL  # next weekly run re-places these
```

I liked this approach because it reused the pipeline I already had. I didn't need a separate one-off cleanup script, and the fix went through the same path as every other record.

## Takeaways

- **Never trust positional pairing between a batch request and its response.** Match on an ID the service hands back to you.
- **`zip()` fails silently.** Uneven lists get truncated with no error. If two lists have to line up, check their lengths first.
- **When counts don't match, stop.** Flagging a batch for review costs a little time. Guessing can write confident, wrong data.
- **Give yourself a way to re-queue records.** A sentinel status that the normal job knows how to handle makes repairs a lot simpler.
