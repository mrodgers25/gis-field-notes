# Survey123 calculationMode isn't guaranteed in the web form

*Worked on: October 2026*

I was building an editing form in Survey123 that would be embedded in an Experience Builder app. The job sounded small: staff needed to open an existing record and correct its location when it was wrong.

The form already let people type coordinates in decimal degrees, degrees decimal minutes, or degrees minutes seconds, and turned those text boxes into a point. That worked for new submissions. Editing turned out to be a different story.

## What went wrong

**Only one geopoint can bind to the layer geometry.** I couldn't keep the original map point and add a second "corrected location" geopoint that also wrote to the feature's shape. One geopoint owns the geometry, so whatever approach I picked had to feed that single question.

**The calculated geopoint didn't update on edit.** The geopoint was calculated from the DD/DDM/DMS text boxes. When a record was opened for editing and someone changed the typed coordinates, the point stayed where it was.

**`calculationMode=manual` didn't rescue it.** The documented way to control when a calculation runs is the `calculationMode` parameter. Setting it to manual should give the question a Calculate button so the user can re-run the expression on demand. Simplified, with placeholder names:

| type | name | calculation | bind::esri:parameters |
|------|------|-------------|-----------------------|
| text | lat_dd | | |
| text | lon_dd | | |
| geopoint | location | `concat(${lat_dd}, ' ', ${lon_dd})` | `calculationMode=manual` |

In the embedded web form, that didn't work either. When I went back to Esri's documentation, the calculation modes are only promised in Survey123 Connect and the field app. The web form isn't on that list, so I was relying on behavior nobody had committed to.

## The simpler answer

I stopped trying to drive the geometry from my own text boxes and used what the web form already has: the built-in coordinate entry under the map. Staff can type a location there in degrees, DMS, MGRS, or UTM, and the point moves. No custom calculation, no Calculate button, and it's a supported part of the web form rather than something I was hoping would work.

## What still doesn't update

Moving the point fixed the geometry, but not everything around it:

- **Separate Latitude/Longitude fields** stored on the record still didn't update after the point was moved. They hold whatever was calculated at the time they were last set, so they can quietly disagree with the geometry after an edit.
- **Fields like edit timestamps** never update on their own. If you want to know when a record was last changed, that field needs its own calculation or you need editor tracking turned on for the layer.

So after an edit, the shape can be right while the attribute columns describing it are stale. That's worth knowing before anyone filters or reports on those columns.

## Takeaway

Check where a Survey123 feature is actually supported before building on it. `calculationMode` is documented for Connect and the field app, not promised in the web form, and an embedded editing form is a web form. When the web form already has a built-in way to do the job, like coordinate entry under the map, use it instead of rebuilding it with calculations. And after any edit workflow, check which fields actually change: geometry moving doesn't mean the latitude, longitude, or timestamp fields moved with it.
