# Using an LLM with web search to fill gaps in a point layer

*Worked on: July 2026*

I had a point layer of about 326 bar and restaurant records, and roughly 150 of them were missing at least one link: a website, Yelp, OpenTable, Instagram or Facebook. Filling those by hand meant a lot of tabs, so I looked for a way to automate the search.

## The situation

My first plan was Google Programmable Search. That didn't last: it had dropped its whole-web search option, so it could no longer search the open web the way I needed.

## What I switched to

I moved to the Claude API's server-side `web_search` tool. The model runs the searches itself and also judges whether a result is really that venue. That second part turned out to be the real win. Writing name-and-address matching by hand, with fuzzy comparisons, abbreviations and "The" prefixes, is a rabbit hole. Asking a model "is this page actually this place?" worked better than my matching rules would have.

## One cheap guardrail

I didn't want to take the model's word for everything, so I kept one simple check that costs nothing: the domain has to fit the field. If the model returns a Facebook link for the Yelp field, it gets rejected.

```python
EXPECTED_DOMAINS = {
    "yelp_url": ("yelp.com",),
    "opentable_url": ("opentable.com",),
    "instagram_url": ("instagram.com",),
    "facebook_url": ("facebook.com",),
}

def domain_fits(field, url):
    host = urlparse(url).netloc.lower()
    allowed = EXPECTED_DOMAINS.get(field)
    if allowed is None:          # website: any domain is plausible
        return True
    return any(host == d or host.endswith("." + d) for d in allowed)

# after the model answers
if url and not domain_fits(field, url):
    url = None                   # treat as not found, note why
```

The website field has no expected domain, so it passes through, and the model's own judgment is the check there.

## What was tricky

Deciding where to trust the model and where not to. I didn't want a pile of validation code that recreated the matching problem I was trying to avoid. A domain check is easy to reason about, never wrong about what it checks, and catches a real failure mode (the right-looking link in the wrong field). Everything else I left to the model's judgment, and to review before anything gets written, which I cover in a separate note.

## The takeaway

- When a search API is the bottleneck, a model with a built-in search tool can replace both the search and the "is this the right entity?" logic.
- Let the model do the fuzzy judgment, and add only cheap, deterministic guardrails for the mistakes you can name.
- Check the output against what the field is supposed to hold. A domain check takes a few lines.
- Don't write model results straight into a live layer without a review step.
