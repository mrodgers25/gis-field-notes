# An environment variable that exists but is blank

*Worked on: September 2026*

I had two small Lambdas whose only job was to send email notifications through SES. Both read their sender address and recipient list from environment variables, with a default in the code as a safety net. One day the jobs ran, reported no errors, and nobody got anything. The job had silently run with zero recipients.

## What went wrong

The code looked reasonable:

```python
import os

SENDER = os.environ.get("SES_SENDER", "notifications@example.org")
RECIPIENTS = os.environ.get("SES_RECIPIENTS", "team@example.org").split(",")
```

My mental model was "if the variable isn't set, use the default." But the keys *were* set. They had been created in the Lambda console with empty values, presumably as placeholders to fill in later. So `os.environ` contained `SES_RECIPIENTS` with a value of `""`, and `.get()` happily returned that empty string.

That's the trap: `dict.get(key, default)` only falls back when the key is **missing entirely**. A key that exists with a blank value is, as far as Python is concerned, a perfectly good value. The default never kicked in.

And because an empty string doesn't throw anything on its way through, nothing complained. The job had no one to send to and finished "successfully."

## The fix

The immediate fix was operational: fill in the real values in the console. Both functions started sending right away.

The code fix is a one-word change. Use `or` instead of the second argument to `.get()`, so both "unset" and "blank" fall through to the default:

```python
SENDER = os.environ.get("SES_SENDER") or "notifications@example.org"
RECIPIENTS = (os.environ.get("SES_RECIPIENTS") or "team@example.org").split(",")
```

`os.environ.get("X")` returns `None` when the key is missing and `""` when it's blank. Both are falsy, so `or` covers both cases.

If a variable is required and there's no sensible default, I'd rather fail loudly than send to a hard-coded fallback:

```python
def require_env(name):
    value = (os.environ.get(name) or "").strip()
    if not value:
        raise RuntimeError(f"Environment variable {name} is missing or blank")
    return value
```

The `.strip()` also catches a value that's just a stray space, which is easy to paste into a console field without noticing.

## Takeaway

"Exists" and "has a usable value" are different things. Lambda console placeholders, `.env` files with `KEY=` lines, and CI settings with empty secrets all produce keys that are present but blank. When reading configuration, treat blank the same as missing: use `os.environ.get("X") or default`, or validate and raise. And for anything that sends notifications, a zero-recipient run is a failure worth surfacing, not a quiet success.
