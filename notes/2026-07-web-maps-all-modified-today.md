# Every web map in my account said "Modified: today"

*Worked on: July 2026*

Right after a bulk `edit_features` run against a hosted layer, I opened my content list and saw that a large number of web maps all showed today's date in the Modified column. My first thought was that my script had touched something it shouldn't have.

## The situation

The script updated attribute values on one hosted feature layer. That's a data edit, nothing else. But 154 web maps in my account showed "Modified: today," and it looked like too much of a coincidence to ignore.

## What looked wrong

The obvious story was that the bulk edit had somehow rewritten every map that touched the layer, and probably some that didn't. If that were true, I'd have a real problem: maps changed without anyone choosing to change them, and no easy way to know what else had shifted.

## How I checked

Instead of guessing, I wrote a read-only diagnostic script. It did no writes at all. For each web map it pulled the map's JSON, checked whether any operational layer referenced the edited layer's item or service URL, and printed the modified date next to the answer.

```python
from arcgis.gis import GIS

gis = GIS("home")
TARGET_URL = "<edited layer service url>"

for item in gis.content.search("", item_type="Web Map", max_items=500):
    data = item.get_data() or {}
    layers = data.get("operationalLayers", []) + data.get("tables", [])
    uses_target = any(TARGET_URL in (l.get("url") or "") for l in layers)
    print(item.title, item.modified, "references edited layer:", uses_target)
```

The result: 152 of the 154 maps didn't reference the edited layer at all. So the edit couldn't have been the cause for those. Editing a layer's attribute data doesn't change a web map's own JSON, because a map stores a pointer to the layer plus its display settings, not the features themselves.

## The takeaway

- Editing the data behind a layer doesn't modify the web maps that display it. The map item's Modified date tracks changes to the map's own definition.
- When something looks like your script's fault, test the causation before you act on it. A cheap read-only script that compares "what changed" against "what could my code have reached" answers the question in minutes.
- Build diagnostics as read-only by default. Being able to run one with zero risk makes you willing to run it right away, instead of cleaning up first and losing the evidence.
- Correlation in timing is weak evidence. Here the dates lined up with my run, but the reference check showed the edit had no path to nearly all of those maps.

I didn't need to roll anything back. The point of the exercise was to avoid "fixing" a problem my script never caused.
