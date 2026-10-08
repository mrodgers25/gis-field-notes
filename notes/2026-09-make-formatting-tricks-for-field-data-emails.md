# Make.com formatting tricks for field-data emails

*Worked on: September 2026*

I was building a Make scenario that turns field-survey submissions into plain-language emails. The data came in as raw form output, which is fine for a database and awkward for a person reading an inbox. Three small problems kept coming up, and each has a short fix I now reach for by default.

## Problem 1: multi-select values with a leading comma

Multi-select answers arrived as a single comma-separated string. In some cases the string started with a comma, so the email read like ", Plumbing, Electrical". It looks sloppy, and it's the kind of thing a recipient notices immediately.

The fix is to strip a leading comma (and any space after it) before the value goes into the email body. In a Make text field I used `replace()` with a regular expression:

```
{{trim(replace(1.repairs; "/^,\s*/"; ""))}}
```

`trim()` handles stray whitespace at the ends, and the pattern only matches a comma at the very start, so clean values pass through untouched.

## Problem 2: fields that aren't always populated

Some fields are only filled in when a condition applies. When they were empty, the email either showed a blank line or, worse, a dangling label like "Notes:" with nothing after it. `ifempty()` gives the field a readable fallback:

```
Notes: {{ifempty(1.notes; "None recorded")}}
```

Choosing the fallback text matters. "None recorded" tells the reader the field was checked and was empty. A blank line leaves them wondering whether something broke.

## Problem 3: coded values that mean nothing to a reader

Survey fields often store codes rather than labels. Nobody reading an email wants to see `3` or `esc_r`. I translated them with `switch()`:

```
{{switch(1.priority; "1"; "Urgent"; "2"; "Soon"; "3"; "Routine"; "Unspecified")}}
```

The last argument is the fallback for any code I didn't anticipate. That means a new or unexpected code produces "Unspecified" rather than an empty string or an error.

## Putting it together

The three combine naturally. For a multi-select field that may be empty, I wrap the cleanup inside `ifempty()`:

```
{{ifempty(trim(replace(1.repairs; "/^,\s*/"; "")); "No repairs selected")}}
```

Order matters here. The cleanup runs first, so a value that was only a comma becomes empty and then correctly falls through to the fallback text.

## Takeaways

- **Format at the last step.** I keep the raw values intact in the data store and only clean them in the email module, so the source of truth stays untouched.
- **Always give lookups a fallback.** A `switch()` with a default turns an unknown code into a visible "Unspecified" instead of a silent blank.
- **Test with the ugly records.** Run the scenario against submissions with empty fields and odd selections, not just the tidy ones. That is where all three of these showed up.
- **Write for the reader.** An email that says "None recorded" is more trustworthy than one that quietly omits a line.
