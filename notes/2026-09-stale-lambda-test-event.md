# The stale Lambda test event that re-ran a two-week-old delta

*Worked on: September 2026*

One manual click in the AWS console re-ran a delta file that was two weeks old. It put records back into a layer that had been deliberately scoped to exclude them, and it overwrote good, current data with stale values. This note covers what happened, how I recovered, and the rule I follow now.

## The situation

The nightly sync between a donor CRM and a hosted feature layer is split into two steps. The first Lambda builds a fresh delta file (what changed since the last run) and drops it in S3. The second step reads that delta and writes it to the layer.

While testing, I had saved a console test event on the write step. The event pointed at a specific staging file in S3, which was the delta from about two weeks earlier. I'd forgotten it was there.

## What went wrong

I ran the saved test event manually. It did exactly what it was told: it applied the old delta. The result:

- About 3,300 out-of-scope records were added to the layer.
- About 4,800 good records were overwritten with stale data from the old file.

Nothing errored. The job reported success, which is the worst kind of failure for a pipeline like this.

## How I recovered

I wrote two small recovery scripts rather than fixing things by hand:

1. **Remove the leaks.** Compare the layer against the current scope rules and delete the roughly 3,300 records that should never have been there.
2. **Restore the overwritten records.** Re-pull the current values for the roughly 4,800 affected records from the CRM and write them back, matching on the stable external key rather than the OBJECTID.

```python
# Sketch: find records that the stale run touched, then re-sync them
stale_keys = set(stale_delta["crm_id"])
current = fetch_from_crm(stale_keys)          # fresh source-of-truth values
payload = [build_update(r) for r in current]  # same builder as the nightly job
apply_updates(layer, payload, key="crm_id")
```

Reusing the same payload builder as the nightly job mattered. A one-off script with its own field mapping is a good way to create a second incident.

## The rule

**Only trigger the step that builds a fresh delta.** The write step should never be something I invoke directly with a hand-picked file. If I need to run the pipeline manually, I start at the beginning, let it produce a new delta, and let the chain continue from there.

A few habits that follow from this:

- Delete saved console test events when a test is done, or name them so they're obviously not for production use.
- Don't point test events at files in the same location the real pipeline reads from.
- Have the write step check the age of its input and refuse a delta older than expected.

## Takeaway

A saved test event is a loaded command waiting for someone to click it. Pipelines that apply changes should only ever be fed by the step that creates those changes, and the write step should be skeptical about stale input.
