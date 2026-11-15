# Arcade can't inject HTML into a Map Viewer pop-up

*Worked on: September 2026*

I was building a pop-up for a business-directory layer. Each business could have social links, a description, and hours, but plenty of records were missing one or more of those. I wanted each section to appear only when it had content, so an empty "Hours" heading wasn't sitting there with nothing under it.

## What went wrong

My first instinct was to let Arcade do the whole job: write an expression that checks the field and returns a full block of HTML (heading, link, formatting) or an empty string. Then I'd drop that expression into Map Viewer's HTML text element.

It doesn't work. Map Viewer escapes the string an expression returns, so instead of rendered HTML the pop-up showed the literal tags as text. Arcade can supply values to the pop-up, but it can't inject markup into it.

## The pattern that works

Flip it around. Put the HTML in the text element as static wrappers, and have each expression return only a CSS display value, `'block'` or `'none'`. The wrapper uses the expression in its `style` attribute:

```html
<div style="display:{expression/show_hours}">
  <b>Hours</b><br>
  {hours}
</div>
```

And the expression is tiny:

```arcade
// show_hours
var h = Trim(DefaultValue($feature.hours, ""));
return IIf(h == "", "none", "block");
```

Each section gets its own expression, and the HTML stays readable because it lives in one place.

## Other traps along the way

None of these threw a helpful error, so they cost more time than they should have:

- **A duplicate `var` declaration made an expression fail silently.** I had declared the same variable twice while copying logic between expressions. The expression just stopped producing output. If a section disappears for every feature, check for that first.
- **`Attachments()` returned no `url` field in this environment.** I had hoped to pull an image link from attachments, and the field simply wasn't there. Don't assume every property in the docs is available everywhere; return the object and look at it before building on it.
- **Expressions only register if you insert them through the field picker.** Typing `{expression/show_hours}` by hand into the HTML didn't wire it up. Inserting it from the picker did, and after that I could move the reference where I wanted it.
- **Expression names are case-sensitive.** `{expression/Show_Hours}` and `{expression/show_hours}` are not the same thing. I settled on all-lowercase names to stop tripping over this.

## Takeaways

- **In Map Viewer pop-ups, Arcade returns values, not markup.** Keep the HTML static and let expressions toggle `display`.
- **Small, single-purpose expressions are easier to debug** than one expression that builds a whole section.
- **Silent failures need a checklist.** Duplicate `var`, missing properties, hand-typed references, and name case: I now check those four before I start doubting the logic.
