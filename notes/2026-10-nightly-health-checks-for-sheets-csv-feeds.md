# Nightly health checks for Google Sheets CSV data feeds

*Worked on: October 2026*

A water-quality nonprofit's website pulls its monitoring data from Google Sheets "export?format=csv" links. When I looked, two of the original five links were failing and nobody had noticed. I built a small nightly check so a broken feed gets reported instead of sitting quietly.

## The tricky part

A plain status-code check isn't enough. A broken Google Sheets export can still return HTTP 200, with an HTML sign-in or error page where the CSV should be. A monitor that only asks "did I get a 200?" would report everything healthy while the site displayed nothing.

So each link has to pass several checks:

1. The response is HTTP 200.
2. The body isn't HTML.
3. The body parses as CSV.
4. There's a header row and at least one data row.
5. Optionally, a list of required columns is present.

## What I built

A small Python AWS Lambda using only the standard library and boto3. With no third-party dependencies there's no zip to build, and I pasted the code straight into the console. EventBridge Scheduler triggers it at 2:00 AM Pacific.

```python
import csv, io, urllib.request

FEEDS = {
    "site-readings": {"url": "https://example.com/export?format=csv", "required": ["date", "value"]},
    # adding a feed is one more line here
}

def check(name, cfg):
    with urllib.request.urlopen(cfg["url"], timeout=20) as r:
        if r.status != 200:
            return f"HTTP {r.status}"
        body = r.read().decode("utf-8", errors="replace")
    if body.lstrip().lower().startswith(("<!doctype html", "<html")):
        return "Got an HTML page instead of CSV"
    rows = list(csv.reader(io.StringIO(body)))
    if len(rows) < 2:
        return "Needs a header row and at least one data row"
    missing = [c for c in cfg.get("required", []) if c not in rows[0]]
    return f"Missing columns: {missing}" if missing else None  # None = pass
```

If anything fails, it sends a styled HTML email through SES that lists broken links in red and passing ones in green. If everything passes, it sends nothing and just writes to the log. A quiet inbox means all feeds are healthy.

The monitor started with 5 feeds and now covers 8. Adding one is a one-line change to the dictionary.

## Common causes of a broken link

When a link fails, these are the usual suspects, so I listed them for whoever gets the alert:

- The sheet's sharing isn't set to "Anyone with the link."
- A tab's gid changed because the tab was deleted and recreated.
- An uploaded .xlsx was re-uploaded and got a new file ID.

## Takeaway

For anything that "returns 200", check what came back, not only that something did. And for a monitor like this, email only on failure. A daily "all good" message gets ignored, while a red list of broken links gets read.
