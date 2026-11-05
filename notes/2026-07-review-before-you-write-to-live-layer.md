# Review before you write to a live layer

*Worked on: July 2026*

I had an enrichment workflow that finds missing links (website, reservation page, social profiles) for a point layer and writes them back to a hosted feature layer. Searching is cheap to redo. Writing to a live layer is not. So I split the workflow into two commands with a human checkpoint between them.

## The situation

The layer is live, and other people's maps read from it. A bad value written by a script isn't an error; it's a wrong link on a public map. I wanted every change to pass in front of my eyes first, and I wanted the search step to be safe to run as often as I liked.

## How it works

**`search`** never touches the layer. For every record and field it writes a row to a review CSV, including a confidence score and notes, whether or not it found anything. Seeing the "not found" rows with their notes matters as much as seeing the hits.

**`apply`** reads the reviewed results, prints the full pending diff (record, field, old value, new value), and asks y/N. Only a "y" reaches `edit_features`.

```python
def apply(changes, layer):
    for c in changes:
        print(f"{c['record']}  {c['field']}:  {c['old']!r} -> {c['new']!r}")
    if input(f"\nWrite {len(changes)} changes to the live layer? [y/N] ").strip().lower() != "y":
        print("Nothing written.")
        return
    layer.edit_features(updates=build_updates(changes))
```

## Keeping reruns cheap and safe

- **A cache keyed by (record, field).** A rerun skips finished work, so I never pay twice for the same lookup.
- **`--limit` and `--max-queries`.** Hard caps on how many records and how many paid calls a run can use.
- **A 5-record test run first.** Before the full batch I run `search --limit 5`, read the CSV, and check that the confidence scores and notes look sensible. Problems in the prompt or the matching logic show up in five rows, not hundreds.

## What was tricky

The temptation is to merge the two steps once you trust the search. I kept them apart because the diff printout is the only place I see the old and new values side by side, and that is where oddities jump out, like a value that's technically a URL but obviously belongs to a different business.

## Takeaway

For anything that writes to a live layer: make the read-and-decide step write to a file, make the write step print exactly what it will change, and require an explicit yes. Add caching and spend caps so the safe path is also the cheap path, and always do a tiny test run before the full batch.
