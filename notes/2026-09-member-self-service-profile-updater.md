# Letting members edit their own profiles without database access

*Worked on: September 2026*

A membership-style project needed people to keep their own profile information current: contact details, a short bio, and a photo. The main data lived in a hosted feature layer that drives a public-facing map, and I didn't want members anywhere near it. Giving a few hundred people edit access to the production layer, or building a custom portal, was far more than the problem called for.

## The approach

I split it into two pieces: a Survey123 form that members use, and a scheduled ArcGIS Online notebook that moves approved changes into the main layer.

**1. A "profile updater" survey.** On login, the survey loads the signed-in member's existing record, so the form opens pre-filled with what we already have. The member changes whatever is out of date and submits. Submissions land in a separate, simple "updates" layer, not the main one.

**2. An hourly notebook.** A scheduled AGOL notebook reads new rows from the updates layer, matches each one to the member's record in the main layer, and writes the changed values across. Photos come along too, since they are attachments and need to be copied separately from the attributes.

The main layer is only ever written by the notebook, running under one service identity. Members only ever touch the updates layer through the survey.

## What was tricky

Attributes and attachments are two different operations. Copying the fields is one `edit_features` call; copying the photo means reading the attachment from the updates row and adding (or replacing) it on the target feature.

The matching key matters. I match on a stable member identifier stored on both layers, never on OBJECTID, which can change if a layer is republished.

I also only apply fields that were actually filled in, so a member who updates their phone number doesn't accidentally blank out their bio.

## A sketch of the sync

```python
from arcgis.gis import GIS

gis = GIS("home")
updates = gis.content.get(UPDATES_ITEM).layers[0]
main = gis.content.get(MAIN_ITEM).layers[0]

new_rows = updates.query(where="processed IS NULL", out_fields="*")

for row in new_rows.features:
    a = row.attributes
    target = main.query(where=f"member_key = '{a['member_key']}'").features
    if not target:
        continue  # log it, don't guess

    feat = target[0]
    changes = {k: a[k] for k in EDITABLE_FIELDS if a.get(k) not in (None, "")}
    feat.attributes.update(changes)
    main.edit_features(updates=[feat])

    # copy the newest photo, if there is one
    # (read the attachment from `updates`, then add it to `main`)
```

After a row is applied, the notebook flags it as processed so the next hourly run skips it.

## Takeaways

- **Separate the thing people can edit from the thing that matters.** A staging layer plus a controlled sync gives members self-service without exposing production data.
- **Pre-filling the form on login is what makes it usable.** People update one field instead of re-entering everything.
- **Treat attachments as their own step.** Syncing attributes doesn't move photos.
- **Only write fields that were provided,** and match on a stable key.
