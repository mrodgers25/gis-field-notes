# A Survey123 geopoint constraint that actually works

*Worked on: September 2026*

After cleaning up survey records that had been submitted sitting on the form's default fallback location, I wanted to stop it from happening again. The goal was simple: if the geopoint is still on the fallback point, the form shouldn't submit. Getting there took one wrong turn first.

## What went wrong

My first idea was to pull the coordinates out of the geopoint with `selected-at()`, treating the geopoint value like a space-separated list. To see what I was working with, I put it into a calculate and a note in Survey123 Connect. Instead of a number, the note showed:

```
[object Object]
```

In Connect's calculate/note context, the geopoint wasn't coming through as a plain string I could split, so `selected-at()` gave me nothing usable to build a constraint on.

## What worked

The function that does this cleanly is `pulldata('@geopoint', ...)`. Given the geopoint question, it returns individual properties:

- `'x'` returns longitude
- `'y'` returns latitude
- `'accuracy'` returns the reported accuracy

For a quick check, a calculate plus a note makes the values visible while testing:

| type | name | label | calculation |
|------|------|-------|-------------|
| geopoint | location | Location | |
| calculate | loc_x | | `pulldata('@geopoint', ${location}, 'x')` |
| note | loc_x_note | Longitude: ${loc_x} | |

For the constraint itself, I used the fact that every location in this project has a positive longitude. The fallback point didn't, so anything that hadn't been moved off it fails:

| type | name | label | constraint | constraint_message |
|------|------|-------|------------|--------------------|
| geopoint | location | Location | `pulldata('@geopoint', ., 'x') > 0` | Tap "Locate Me" to capture your location before submitting. |

The `.` refers to the current question, so the constraint reads the geopoint's own longitude. This is deliberately specific to this project's geography. It isn't a general "is this a real location" test, it's "is this on the right side of the line where all our real locations are."

## The accuracy constraint I couldn't test

I also considered an accuracy constraint, something like:

```
pulldata('@geopoint', ., 'accuracy') < 100
```

On a real device capturing a GPS fix, that makes sense: it would reject a fix that's too rough to trust. The problem is testing it. In Connect's preview, and when you set the point by clicking on the map instead of capturing a fix, there's no real device accuracy to evaluate, so there's no way to confirm the constraint behaves as intended before it reaches field workers.

## Takeaways

- **Use `pulldata('@geopoint', ...)` to get at geopoint parts.** `'x'` is longitude, `'y'` is latitude, `'accuracy'` is accuracy. Don't try to split the geopoint with `selected-at()`.
- **Surface values in a note while you build.** Seeing `[object Object]` told me right away that approach was a dead end.
- **Use what you know about your data.** A simple sign check on longitude was enough to catch the fallback point for this project's locations.
- **Be honest about what you can test.** An accuracy check is reasonable for real devices, but if your test environment can't exercise it, treat it as unverified.
