# Make.com formulas that fail without an error

*Worked on: September 2026*

I had a Make scenario that turns field-inspection submissions into an email. The email body held several nested inline formulas: show this sentence if a condition is met, show that label if a field has a value, and so on. They kept failing, and they failed without telling me. No error, no red bubble on the module. The email just came out with the wrong text, or with raw function text printed where a value should have been.

It turned out to be three separate problems.

## 1. The editor changed my empty strings

I was comparing fields against an empty string, `""`. When I pasted a formula into the editor, that `""` became `""""`. Likely autocomplete or smart punctuation. Either way, the comparison no longer checked for "empty". It checked against a string of quote characters, which never matched, so the condition quietly went the wrong way every time.

## 2. There's no `and` operator

I wrote conditions the way I would in most languages, joining two tests with `and`. Make's formula language has no `and` operator. When you need two conditions to both be true, you nest one `if()` inside another:

```
{{if(1.status = "fail"; if(1.followup = "yes"; "Follow-up required"; "Failed, no follow-up"); "Passed")}}
```

That works, but a few levels of it inside an email body gets hard to read and harder to debug.

## 3. Set Multiple Variables needs `{{ }}` too

When I started moving logic out of the email, I assumed the variable value field would evaluate functions on its own. It doesn't. Without the `{{ }}` wrapper, the function is printed as literal text, so the variable holds the words `ifempty(...)` instead of the result. Same wrapping rule as everywhere else in Make.

## The fix: compute first, then format

I moved the logic into a **Set Multiple Variables** module placed just before the email module. Each computed value gets its own named variable, and the email body only references those variables.

This helped in two ways:

- **Every value shows up in the execution history.** I can open a run and see exactly what each variable resolved to. When something is off, I know which one, instead of staring at a finished email and guessing which nested formula went wrong.
- **The email body becomes plain.** It's mostly text with variable references, so there's much less to mistype or have the editor rewrite.

I also stopped comparing against empty strings. Instead, I give a possibly-empty value a sentinel like `"none"` up front, then compare against that:

```
followup_note   = {{ifempty(1.followup_note; "none")}}
show_followup   = {{if(followup_note = "none"; "no"; "yes")}}
```

A sentinel is a normal word. It doesn't get mangled on paste, and I can see it in the execution history. An empty string looks the same as a value that never got set.

## Takeaways

- **Silent failures are the dangerous ones.** If a formula can be wrong without an error, check its output, not just whether the module ran.
- **Use an intermediate step you can inspect.** A Set Multiple Variables module before the output module turns hidden logic into something you can see in every run.
- **Avoid `""` comparisons.** Use a visible sentinel value instead.
- **Remember the wrappers and the missing `and`.** Wrap functions in `{{ }}` wherever they appear, and nest `if()` for compound conditions.
