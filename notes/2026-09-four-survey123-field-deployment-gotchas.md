# Four Survey123 field-deployment gotchas

*Worked on: September 2026*

A survey can pass every test in Survey123 Connect and still trip up once it's on real phones, launched from real pop-ups, with real scanners and keyboards. During one field deployment I hit four separate problems in quick succession. None of them threw an obvious error, so I'm writing them down together.

## 1. A read-only default that overrode the URL

The survey was launched from a Field Maps pop-up link that passed the country in the URL, so field workers wouldn't have to pick it. But the form kept showing the wrong country.

The cause: the country question was set to read-only and had a hardcoded default. The default won over the value coming in from the URL, so every record got the same country no matter which feature the link came from.

The fix was to drop read-only and use the `hidden` appearance instead. The question still receives the URL value; it just isn't shown.

| type | name | label | appearance | default |
|------|------|-------|------------|---------|
| text | country | Country | hidden | |

The link itself passes the value as a field parameter, along the lines of `...?field:country={country}`.

## 2. A barcode scanner that added asterisks

The barcode scanner wrapped scanned values in asterisks, so a value that should have matched existing records came in with an extra `*` at each end. Rather than ask field workers to fix it by hand, I cleaned it in a calculation with `translate()`, which replaces characters, here swapping `*` for nothing:

| type | name | calculation |
|------|------|-------------|
| barcode | barcode_raw | |
| calculate | barcode | `translate(${barcode_raw}, '*', '')` |

Downstream logic then uses the cleaned `barcode` field, not the raw scan.

## 3. A phone keyboard fighting regex validation

One field was digits-only, enforced with a regex constraint. On a phone it kept failing, and the culprit was the keyboard: SwiftKey was interfering with what got typed into the field, so input that looked right didn't pass the regex.

Setting the `numbers` appearance on the question fixed it. That brings up the numeric keypad instead of the full keyboard, which keeps the keyboard's extras out of the way:

| type | name | appearance | constraint |
|------|------|------------|------------|
| text | id_number | numbers | `regex(., '^[0-9]+$')` |

## 4. select_one_from_file that just stopped working

A `select_one_from_file` question that had been working stopped working. What brought it back was fully resaving the survey in Survey123 Connect. If a choice list from a file goes quiet, a full resave in Connect is worth trying before you start rebuilding anything.

## Takeaways

- **Use `hidden`, not read-only, for values passed in by URL.** A read-only question with a default can override what the link sends.
- **Clean scanner input in a calculation.** `translate()` strips characters like stray asterisks before anything depends on the value.
- **Pick the keyboard with the appearance.** `numbers` on a digits-only field avoids keyboard apps interfering with regex validation.
- **When a file-based choice list breaks, resave in Connect.**
- **Test on the devices and launch paths field workers actually use.** All four of these only showed up in deployment.
