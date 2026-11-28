# Archiving rows from a visually grouped sheet

*Worked on: October 2026*

I was building a Lambda that tidies up a task-tracking sheet: any task marked Done gets moved to an archive tab so the main tab only shows open work. Moving a row sounds simple. Copy it to the archive, delete it from the main tab. The sheet's layout made that unsafe.

## What was tricky

The sheet is grouped visually by client. Each client's name appears on only one row, the first row of their block, and the rows below it are left blank in the client column so the sheet reads like an outline:

| Client | Task | Status |
|--------|------|--------|
| Client A | Task 1 | Done |
| | Task 2 | Open |
| | Task 3 | Open |
| Client B | Task 4 | Open |

That's easy for a person to read, but the client name only lives on one row. If the Done task happens to be the labeled row, a naive archive deletes the only copy of the client's name. Task 2 and Task 3 are still there, but now they look like they belong to whoever is above them, or to nobody. Nothing errors. The sheet just quietly loses information.

There was a second case. If a client's *only* task is Done, deleting it removes the client from the sheet entirely, which isn't what anyone wants. The client is still active; they just don't have open work right now.

## How I solved it

Before deleting a labeled row, the Lambda checks whether any rows in the same client block survive. If they do, it copies the label onto the first surviving row. If none survive, it doesn't delete the row at all. It converts it into a placeholder row that keeps the client name and clears out the task details.

```python
# Only these columns are cleared when a row becomes a placeholder.
TASK_DETAIL_COLUMNS = ["task", "status", "due", "notes"]

def plan_archive(block):
    """block: list of row dicts for one client, first row holds the label."""
    label = block[0]["client"]
    done = [r for r in block if r["status"] == "Done"]
    keep = [r for r in block if r["status"] != "Done"]

    if not done:
        return [], block

    if keep:
        # Move the label onto the first row that will survive.
        keep[0]["client"] = label
        return done, keep

    # Client's only task(s) are Done: keep a placeholder row instead.
    placeholder = dict(block[0])
    for col in TASK_DETAIL_COLUMNS:
        placeholder[col] = ""
    return done, [placeholder]
```

The important part is that the plan is worked out per client block first, and only then applied to the sheet. Deciding row by row while deleting is how you end up with shifted indexes and labels copied to the wrong place.

## Testing it

I tested four scenarios before letting it touch the real sheet: the Done task being an unlabeled row, the Done task being the labeled row with other tasks surviving, a client whose only task was Done, and a mix across clients. The tests paid off. One of them caught the placeholder conversion blanking a column it shouldn't have. My first version cleared every column except the client name, and one of those columns needed to stay. The fix was to clear an explicit list of task-detail columns instead of "everything but the label."

## Takeaway

When a sheet is formatted for people, the formatting is often carrying data. Blank cells that mean "same as above" are a dependency between rows, and any automation that deletes rows has to respect it. Plan the change per group, move the label before you delete its row, never let a group disappear entirely, and be explicit about which columns a cleanup is allowed to clear. Then test the edge cases on a copy, because the bug that blanks the wrong column won't announce itself.
