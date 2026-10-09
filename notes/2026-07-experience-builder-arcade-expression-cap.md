# Experience Builder's 10-Arcade-expressions-per-page cap

*Worked on: July 2026*

I was building an Experience Builder app where each selected feature needed to show a handful of images. The page had 16 image widgets, and my first instinct was the one I'd use in a popup: give each image widget its own Arcade expression to pick the right URL, with a fallback when nothing was selected.

## What went wrong

Experience Builder limits a page to 10 Arcade expressions. With 16 image widgets, each wanting its own expression, I ran out before I'd wired up even two-thirds of the page. Nothing about the individual expressions was wrong; the page as a whole was over budget.

Trimming wasn't an option. All 16 images were needed, and merging them into one expression would have meant one big dynamic string that the widgets couldn't each consume separately.

## How I solved it

I stopped using Arcade for the images at all. Instead:

1. **Bind each image widget directly to an attribute** on the data source (the field that holds the image link). Plain attribute binding doesn't count against the Arcade cap.
2. **Use "View for empty selection"** on each widget to define what it shows when nothing is selected. Each image reverts to a default instead of going blank or showing a broken icon.

The Arcade I'd been about to write for every widget was essentially this:

```javascript
// What I was going to repeat 16 times
IIf(IsEmpty($feature.photo_1), "default_image_url", $feature.photo_1)
```

The fallback logic in that expression is exactly what the empty-selection state already does, and the attribute binding replaces the field lookup. So the expression added nothing the widget couldn't already do natively.

## Takeaways

- **Check the per-page limits early.** The cap is on the page, not the widget, so a design that is fine at 6 widgets can quietly stop working at 16.
- **Reach for Arcade last in Experience Builder.** If the value is already an attribute and the only logic is "show a default when empty," the built-in binding and empty-selection view cover it.
- **Reserve your expressions for real calculations.** Keeping the 10 for things that genuinely need computing (formatted text, conditional labels) leaves room for the page to grow.
- **Look at the empty state as a design decision.** Setting the default image deliberately made the app look finished before anyone selected anything.
