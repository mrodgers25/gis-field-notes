# "No mapping exists from object type MapPoint": two geopoints in one form

*Worked on: October 2026*

A needs-assessment survey I was reviewing failed every time someone tried to submit it. The error was:

```
No mapping exists from object type MapPoint to a known managed provider native type.
```

That message doesn't point at any particular question, so it took some digging to see what was going on.

## What went wrong

The form had two geopoint questions. The first one was a helper, marked with `bind::esri:fieldType = null` so it wouldn't get its own field. The second was the real location question.

The problem is that Survey123 Connect uses the **first** geopoint in the XLSForm as the layer's geometry. Here that meant the helper became the geometry, even though it was marked as not-stored, and the real location question was left with nowhere to go. On submit, Survey123 tried to write a point value into a regular attribute field, and the service rejected it with the MapPoint error.

Simplified, the problem layout looked like this:

| type | name | bind::esri:fieldType |
|------|------|----------------------|
| geopoint | helper_point | null |
| geopoint | location | |

## The fix

Move the real geopoint so it comes first in the form, then republish.

| type | name | bind::esri:fieldType |
|------|------|----------------------|
| geopoint | location | |
| geopoint | helper_point | null |

Two things to plan for:

1. **It's a schema change.** Changing which question holds the geometry isn't something Survey123 can patch into the existing layer, so republishing creates a new feature service. Any maps, views, or dashboards using the old one need to be repointed.
2. **Clear the stale form on the device.** The field app had cached the old version. Until that copy was deleted and the form downloaded again, the device kept submitting against the broken layout.

## Other things the same review found

While I was in the XLSForm, two more problems turned up. Neither threw an error, which made them worse.

- **Curly quotes in `instance_name`.** The expression had been typed or pasted with smart quotes (`“ ”`) instead of straight quotes (`" "`). XLSForm expressions need straight quotes. The fix was retyping them.
- **A trailing space in a question name.** One `name` cell had a space at the end. Other questions' `relevant` expressions referenced it as `${question}`, which didn't match `question `, so those questions never showed up. Nothing failed, the logic just quietly never fired.

A quick check I now run on the survey sheet before publishing:

```python
import pandas as pd

survey = pd.read_excel("form.xlsx", sheet_name="survey")

# question names with leading/trailing whitespace
names = survey["name"].dropna().astype(str)
print(names[names != names.str.strip()])

# curly quotes anywhere in the sheet
curly = survey.astype(str).apply(lambda c: c.str.contains("[“”‘’]"))
print(survey[curly.any(axis=1)][["type", "name"]])

# geopoint order: the first one becomes the geometry
print(survey[survey["type"] == "geopoint"][["name", "bind::esri:fieldType"]])
```

## Takeaway

In a Survey123 form with more than one geopoint, order matters: the first one becomes the geometry, whatever else you've marked on it. Put the real location question first from the start, because fixing it later means a new service and clearing devices. And when a form misbehaves without any error, look for invisible problems like smart quotes and stray spaces before blaming the logic.
