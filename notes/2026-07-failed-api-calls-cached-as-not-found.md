# Failed API calls cached as "not found"

*Worked on: July 2026*

I had a script that fills in missing attributes on a hosted feature layer by calling a search API. It keeps a cache so a rerun doesn't pay for the same lookup twice. That cache turned out to be quietly lying to me.

## The situation

The script searches for a value for each record and field, then caches the answer. Reruns skip anything already in the cache. When the API's credits ran out partway through a run, the calls started failing, and my code treated "the API call failed" the same as "I searched and found nothing." Both got cached as "not found."

## What was tricky

Nothing crashed, and the output looked plausible: a lot of "not found" results. I only got suspicious because of how many there were. Of 418 "not found" entries, 371 were really failures.

The giveaway was timing. Real web-search calls took several seconds each. The bad entries came back less than 1.5 seconds apart, in solid blocks. A genuine "searched and found nothing" doesn't look like that. An instant rejection does.

## How I fixed it

Three changes:

1. **A distinct error type.** A failed call raises its own exception, and that exception never reaches the cache. Only a real search that found nothing gets stored as "not found."
2. **A circuit breaker.** After 5 consecutive failures the run stops, instead of burning through the whole list and caching garbage.
3. **A targeted cleanup.** Rather than a blanket `--force` rerun, which would repeat the lookups that had worked and cost credits, I found the bad entries and removed only those.

```python
class SearchFailed(Exception):
    """The call itself failed. Never cache this."""

def lookup(record, field):
    try:
        result = call_search_api(record, field)
    except Exception as exc:
        raise SearchFailed(str(exc)) from exc
    return result  # may legitimately be None: searched, found nothing

consecutive_failures = 0
for record, field in work_items:
    try:
        cache[(record, field)] = lookup(record, field)
        consecutive_failures = 0
    except SearchFailed:
        consecutive_failures += 1   # not cached, so it retries next run
        if consecutive_failures >= 5:
            raise SystemExit("5 failures in a row, stopping")
```

For the cleanup, I identified the suspect entries by the timing pattern, deleted just those keys from the cache, and let the next normal run redo them.

## The takeaway

- "The call failed" and "the answer is nothing" are different outcomes. Give them different types in code, and only cache the second.
- Fail fast. A handful of consecutive errors means something systemic, like credits or auth, and you should stop.
- If results look too empty, check the timing. Suspiciously fast, evenly spaced responses are a sign the work never happened.
- When cleaning up, fix the bad entries instead of rerunning everything with `--force`.
