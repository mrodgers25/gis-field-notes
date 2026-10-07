# A Survey123 calculated field that silently dropped selections

*Worked on: September 2026*

A repair-checklist form I was working on used a multi-select question, and I had a calculated field that built a single readable string of everything the technician checked. That string fed the downstream reports. Then I noticed that some submissions were missing checked items in that string, with no error anywhere.

## What went wrong

The calculated field used `concat()` to assemble the list. On paper it looked right, but in practice some of the selections that were checked in the form did not show up in the resulting text. Nothing failed: the submission saved, the field was populated, it just wasn't the whole story. That is the worst kind of bug in field data, because the record looks complete.

I didn't find a clean explanation for exactly why those items were lost, and I didn't want to build a pipeline on top of a field I couldn't trust. So rather than keep patching the expression, I stopped treating the concatenated string as the source of truth.

## The fix: rebuild the list downstream

Each checklist item also exists as its own boolean field on the layer. Those individual values were reliable. So I changed the downstream logic to ignore the concatenated field and rebuild the list from the booleans every time it was needed.

In Python, that looked like this:

```python
CHECKLIST = {
    "roof_leak": "Roof leak",
    "plumbing": "Plumbing",
    "electrical": "Electrical",
    "flooring": "Flooring",
}

def build_repair_list(attrs: dict) -> str:
    items = [label for field, label in CHECKLIST.items()
             if attrs.get(field) in (1, True, "yes", "Yes")]
    return ", ".join(items)
```

The same idea works in Arcade for a popup or dashboard, using an `IIf` per field and filtering out the empties before joining.

A nice side effect: the label mapping now lives in one place, so changing wording for a checklist item doesn't require editing a long expression inside the form.

## Takeaways

- **A derived text field is a convenience, not a record.** Keep the atomic values (one boolean per option) and build display strings from them.
- **"No error" doesn't mean "correct."** I only caught this because I compared a few submissions against what the technician actually checked. Spot-checking a handful of records against the raw inputs is cheap insurance.
- **If a calculation can't be trusted, move it out of the form.** Doing the assembly downstream, where I can test and log it, made the problem visible and fixable.
