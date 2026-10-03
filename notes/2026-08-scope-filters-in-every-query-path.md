# Scope filters belong in every query path

*Worked on: August 2026*

On a CRM-to-map sync for a nonprofit client, the map was only supposed to show active donors. The initial backfill respected that. The nightly incremental job didn't, and staff and volunteers started showing up on the layer.

## The situation

The pipeline had two ways of pulling records from the donor CRM:

1. A one-time backfill that loaded the full history.
2. A nightly incremental query that picked up whatever changed since the last run.

I wrote the backfill first and gave it a filter to limit it to active donors. When I wrote the nightly query later, I wrote it as a "what changed since yesterday" query and never copied the scope rule over.

## What was tricky

Nothing failed. The nightly job ran on schedule, the counts looked plausible, and the layer kept updating. The only sign was people on the map who didn't belong there, because staff and volunteers live in the same CRM as donors and any of them can have an edit in a given day.

The bug wasn't in either query on its own. Each one was correct for what I had in mind when I wrote it. The problem was that the scope rule existed as a copy-pasted `WHERE` clause in one place and as nothing in the other. Whenever a rule is written inline, a new query path can silently skip it.

## How I fixed it

I pulled the scope rule into one shared function that both paths call. Neither query writes its own version of "who counts" anymore, and each only adds its own extra condition, such as the changed-since date.

```python
def scope_clause():
    """The single definition of who belongs on the map."""
    return "record_type = 'Donor' AND status = 'Active'"

def build_query(since=None):
    where = scope_clause()
    if since:
        where += f" AND last_modified >= '{since}'"
    return f"SELECT Id, ... FROM contact WHERE {where}"

backfill_sql = build_query()
nightly_sql = build_query(since=last_run_iso)
```

(The field and object names above are placeholders, not the real schema.)

I also cleaned up the layer afterward by removing the records that should never have been there. A fix that only stops new leaks leaves the old ones visible.

## The takeaway

- A scope rule is business logic, so it should be defined once. Don't paste it into each query.
- Any new way of reading the source, whether a backfill, an incremental job, or a one-off script, should go through the same function.
- When a pipeline has an initial load and an incremental load, compare what each one lets through. They're easy to get out of sync, and neither will raise an error about it.
