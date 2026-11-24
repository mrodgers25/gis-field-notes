# Why a dashboard said it used two web maps

*Worked on: October 2026*

An ArcGIS Dashboard's item page listed two web maps as dependencies. Only one map was visible on the dashboard. Nothing looked broken, but a dependency you can't see is the kind of thing that bites later: someone deletes the "unused" map, or shares the dashboard and forgets the second map, and a widget quietly stops working.

## Why it was tricky

The dashboard doesn't tell you *which* part of it is holding onto the second map. The item's `dependent_upon()` list just says the maps are there. To find the culprit you have to read the dashboard's JSON, and that structure has changed across dashboard versions. Widgets don't always live in one flat list, and there are separate desktop and mobile views, each with its own widgets. Header and sidebar selectors sit in their own sections too. Looking at only the top-level widget list misses things.

## What I built

I wrote a read-only audit notebook. It doesn't change anything in the dashboard. It:

1. Walks the dashboard JSON recursively and collects every widget and selector it finds, wherever it's nested, including the mobile view, header, and sidebar.
2. Resolves each data source back to the web map and layer it actually depends on.
3. Compares that list with what `dependent_upon()` reports, so any map in the dependencies that nothing visible explains stands out.

The recursive walk is the part that makes it hold up across versions. Simplified:

```python
from arcgis.gis import GIS

gis = GIS("home")
dash = gis.content.get("<dashboard item id>")
data = dash.get_data()

def find_components(node, path="root"):
    """Yield every dict that looks like a widget or selector, wherever it is nested."""
    if isinstance(node, dict):
        if "type" in node and ("datasets" in node or "itemId" in node):
            yield path, node
        for key, value in node.items():
            yield from find_components(value, f"{path}.{key}")
    elif isinstance(node, list):
        for i, item in enumerate(node):
            yield from find_components(item, f"{path}[{i}]")

for path, comp in find_components(data):
    print(comp.get("type"), comp.get("name", ""), comp.get("itemId", ""), path)

print(dash.dependent_upon())
```

Printing the path along with each component matters. When a stray map shows up, the path tells you whether it's in the desktop layout, the mobile view, or a header selector, which is most of the way to fixing it.

## The usual culprits

When a dashboard lists a web map you can't see, it tends to be one of these:

- **A widget whose data source still points at a deleted map widget.** The map widget was removed, but a list, indicator, or chart was originally wired to that map's layer and still references it.
- **A separate map in the mobile view.** The mobile layout can have its own map widget, pointing at a different web map than the desktop one.
- **A header or sidebar selector set up on an old map.** Selectors live outside the main layout, so they're easy to forget when the map is swapped.

## Testing it

Before trusting the notebook on a real dashboard, I tested it against a mock dashboard that I'd deliberately set up with a stray map. If the audit couldn't find a dependency I planted on purpose, it wasn't going to find one I didn't know about. Then I wrote it up so it could be reused across organizations, not just for this one dashboard.

## Takeaway

The item page tells you *that* a dashboard depends on something, not *where*. Don't assume the dashboard JSON is flat or that the desktop layout is the whole dashboard: walk the entire structure, including mobile, header, and sidebar, and record where each reference lives. And keep audits like this read-only. The goal is to find the stray reference so a person can decide what to do with it, not to have a script start rewriting a live dashboard.
