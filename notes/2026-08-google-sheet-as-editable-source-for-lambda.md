# A Google Sheet as the editable source for a Lambda pipeline

*Worked on: August 2026*

For a client's Lambda pipeline, I needed a way for non-technical staff to control which items the job processed without asking me to change code and redeploy. A Google Sheet turned out to be the simplest answer.

## The situation

The pipeline had a list of things it should work on, and that list changed over time. Items got added, paused, and retired. Originally the list lived in the Lambda code, so every change meant me editing, packaging, and deploying. That's a poor fit when the person who knows what should run is the client.

## The design

The client edits a sheet. A **Status** column controls what runs: only rows with the right status value get processed on the next run. Pausing something is changing one cell, with no deploy.

The Lambda reads the sheet through a **read-only service account**. The sheet is shared with that account as a viewer, so the pipeline can never modify the client's data, and nothing about the client's own Google account is involved.

## What was tricky

A pipeline that depends on a sheet now depends on something I don't control. The sheet can be unshared, moved, or temporarily unreachable, and a nightly job shouldn't fail or, worse, run with an empty list because it couldn't read the file.

So I kept a **static list in the code as a fallback**. If the sheet can't be read, the Lambda logs it and runs against the list baked into the deployment. The sheet is the source of truth when it's reachable, and the code is the safety net when it isn't.

I was careful to treat "couldn't read the sheet" differently from "the sheet is legitimately empty." An unreachable sheet triggers the fallback. A readable sheet with no active rows means nothing runs, because that's what the client asked for.

```python
def load_active_items():
    try:
        rows = read_sheet_rows()          # service-account read, placeholder
    except Exception as exc:
        print(f"Sheet unreachable, using fallback list: {exc}")
        return FALLBACK_ITEMS

    return [
        r["name"] for r in rows
        if r.get("Status", "").strip().lower() == "active"
    ]
```

Normalizing the Status value (trim and lowercase) matters more than you'd expect. People type things by hand, and a stray space shouldn't silently drop a row.

## The takeaway

- Move the list that changes often out of the code and into something the owner can edit. A sheet is enough for that.
- Give the pipeline read-only access so it can't damage the source.
- Always have a defined fallback for when the external source is unavailable, and make sure "unreachable" and "empty" are handled as two different cases.
- Log which source a run used, so you can tell afterward whether it ran from the sheet or the fallback.
