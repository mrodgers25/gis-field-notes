# The review CSV that hid earlier results

*Worked on: July 2026*

I have an enrichment workflow that searches for missing links on a point layer, writes every result to a review CSV, and only updates the live layer after I've looked at that file. The review step is the whole point of the design, so the day the CSV quietly stopped showing me things was a real problem.

## The situation

The workflow runs in two stages: `search` finds candidate values and `apply` writes the approved ones. Results go into a cache as they come in, and the CSV is what I read to decide what's safe to apply. I run `search` in several passes (a small test batch first, then bigger ones), reviewing in between.

## What went wrong

Each `search` run wrote the review CSV from scratch using only the results of *that run*. Values found in an earlier run, but not yet applied, vanished from the file I was reviewing. They were still sitting in the cache, so nothing was lost, but the file in front of me no longer told the truth about what was pending. If I had trusted it, I'd have applied only the latest batch and assumed the rest didn't exist.

The failure is easy to miss because nothing errors. The CSV is valid and looks complete. It's just a view of one run, presented as the view of everything.

## The fix

The cache is the source of truth, so the CSV should always be a projection of the whole cache, never of the current run. Every run now finishes by rebuilding the CSV from all cached results:

```python
def write_review_csv(cache, path):
    rows = []
    for (record_id, field), result in cache.items():
        rows.append({
            "record": record_id,
            "field": field,
            "value": result.get("value"),
            "confidence": result.get("confidence"),
            "notes": result.get("notes"),
        })
    rows.sort(key=lambda r: (str(r["record"]), r["field"]))
    with open(path, "w", newline="") as f:
        w = csv.DictWriter(f, fieldnames=rows[0].keys())
        w.writeheader()
        w.writerows(rows)
```

I also added a `rebuild` command that does only that: it reads the cache and regenerates the CSV with no API calls and no cost. That's handy any time the file gets out of sync, gets edited by accident, or I just want a fresh look at everything pending.

## The takeaway

- If a file is something you review before a write, make it a deterministic function of your stored state, not a log of the latest run.
- Keep one source of truth (the cache) and treat every output file as disposable and regenerable.
- Give yourself a free, no-side-effects command to regenerate it. If rebuilding costs nothing, you'll do it before every review.
- A review step is only as good as the completeness of what it shows you. Silent omissions are worse than visible errors.
