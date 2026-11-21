# Saving coordinates exactly as the field worker typed them

*Worked on: October 2026*

I was working on a Survey123 form where field workers could enter a location three ways: decimal degrees, degrees decimal minutes, or degrees minutes seconds. Whichever method they picked, the form converted it and stored two things: the final point and the calculated latitude/longitude.

What it didn't store was what the worker actually typed. Once the form was submitted, the raw entry was gone. If a point looked wrong later, there was no way to tell whether the worker had typed something wrong or the conversion had done something unexpected. I wanted the original entry kept alongside the result, just for reference.

## The approach

I added one text question with a `calculation` that builds a readable string from the raw inputs. A nested `if()` on the chosen entry method decides which inputs to concatenate. Simplified, with placeholder question names:

| type | name | calculation |
|------|------|-------------|
| select_one coord_method | coord_method | |
| calculate | coords_as_entered | *(expression below)* |

```
if(${coord_method} = 'dd',
   concat(${lat_dd}, ', ', ${lon_dd}),
if(${coord_method} = 'ddm',
   concat(${lat_d}, '° ', ${lat_m}, "'", ', ',
          ${lon_d}, '° ', ${lon_m}, "'"),
if(${coord_method} = 'dms',
   concat(${lat_d}, '° ', ${lat_m}, "' ", ${lat_s}, '"', ', ',
          ${lon_d}, '° ', ${lon_m}, "' ", ${lon_s}, '"'),
'')))
```

The final `''` covers the case where no method has been chosen yet, so the field stays empty instead of erroring.

## The tricky part: quoting the symbols

The minutes symbol is a single quote (`'`) and the seconds symbol is a double quote (`"`). Those are also the characters XLSForm uses to delimit strings, so writing them naively breaks the expression. Something like `'''` or `"""` just ends the string early and the form won't validate.

The trick is to wrap each symbol in the *other* kind of quote:

- Minutes: put the single quote inside double quotes, `"'"`
- Seconds: put the double quote inside single quotes, `'"'`

Mixing quote styles in one `concat()` looks odd, but XLSForm accepts either style for a string literal, so each piece just needs to be internally consistent. The degree symbol has no such conflict and can sit in ordinary single quotes.

One related thing to watch: these have to be straight quotes. If the expression gets pasted through a word processor or email and picks up curly quotes, it breaks in a way that's hard to spot by eye.

## One field, not many

I considered storing each component (degrees, minutes, seconds for latitude and longitude) in its own field. That would make the raw values filterable and easier to analyze. But that wasn't the point here. The raw entry was for reference, something to look at when a point seemed off, not something anyone would query or symbolize. A single combined text field kept the schema small and the value easy to read in a pop-up or table, so I went with one field.

## Takeaway

If a form converts what people type into something else, consider keeping the original too, so you can trace a bad result back to its input. In XLSForm, when the text you need to output contains quote characters, wrap each one in the opposite quote style. And match the storage to the use: if a value is only for reference, one readable text field is usually enough.
