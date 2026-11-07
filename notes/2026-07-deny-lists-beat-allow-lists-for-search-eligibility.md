# Deny-lists beat allow-lists for search eligibility

*Worked on: July 2026*

I was setting up an enrichment script that searches the web for missing links (website, reservation page, social profiles) on a point layer of bars and restaurants. Before spending a single search credit, I had to decide which records were even eligible. That sounds like a throwaway filter, but the way you write it decides what happens to data you haven't seen yet.

## The situation

The layer has a type field backed by a coded-value domain. Some records shouldn't be searched at all:

- Anything marked closed. The domain had both "CLOSED" and "Closed", so a plain equality check would have missed one of them.
- Non-venue types like Park, Condo and Theater, which have no reservation page or restaurant profile to find.

On top of that, the reservation-platform search should only run for restaurants.

## What was tricky

My first instinct for the restaurant check was an allow-list: a list of cuisine values that count as restaurants. It reads naturally, but it fails quietly. If someone adds a new cuisine to the domain later, an allow-list skips those records and nothing tells me. They just never get searched, and the gap looks like "no results."

## The fix

I flipped it into a deny-list of the types that are *not* restaurants. Anything not on the list is eligible by default, so a new cuisine value gets searched without me touching the code. The failure mode changes from "silently skipped" to "searched when it maybe shouldn't have been." That is a much better failure: it costs a few searches and shows up in the review file, instead of hiding.

```python
NOT_VENUES = {"park", "condo", "theater"}
NOT_RESTAURANTS = {"bar", "nightclub", "brewery"}  # illustrative

def is_closed(rec):
    return (rec.get("status") or "").strip().lower() == "closed"

def eligible(rec):
    t = (rec.get("type") or "").strip().lower()
    return not is_closed(rec) and t not in NOT_VENUES

def eligible_for_reservations(rec):
    t = (rec.get("type") or "").strip().lower()
    return eligible(rec) and t not in NOT_RESTAURANTS
```

Normalizing with `strip().lower()` handles the CLOSED/Closed split in one place.

## The caveat

A deny-list still needs a human look. A nail salon turned up in the layer, which no type list would have predicted. It was a good reminder that the filter is a cost guard, not a guarantee, and that the review step before writing to the live layer is what catches the rest.

## The takeaway

- When a filter decides what gets skipped, choose the version whose mistakes are visible. Deny-lists fail toward "included"; allow-lists fail toward "silently dropped."
- Normalize case and whitespace before comparing domain values, especially when a domain has near-duplicate entries.
- Revisit the type list by eye now and then. Data has surprises that no list anticipates.
