# Open data portals aren't all the same

*Worked on: July 2026*

When a project says "we'll pull the data from the state's open data portal," it's easy to hear that as one known thing. During discovery on a health-data project I learned it isn't. I had a mental default: open data portal means Socrata, so use the SODA API, filter with SoQL, done. California's health data portal turned out to run CKAN instead, and a couple of the official APIs I wanted sat behind authentication.

## What was tricky

Nothing was broken, which is what made it easy to miss. The assumption just doesn't hold, and it shows up as wasted time:

- **Different platform, different API.** CKAN and Socrata both publish datasets, but the endpoints, query style, and response shapes differ. Code written against one doesn't carry over to the other.
- **"Public" doesn't always mean "no credentials."** Some official APIs required auth, so a quick unauthenticated test call wasn't enough to prove an integration was feasible.

## How I handled it

I stopped treating the data source as a detail to sort out during the build and moved it into discovery. For each source, before committing to an approach, I check:

1. **What platform is it?** The page footer, the API docs link, and the URL structure usually give it away. A quick probe settles it.
2. **Is there an API, and does it need a key or token?** If it does, who gets the credentials, and where do they live?
3. **What does a real request return?** Not the docs, an actual call.

A tiny probe like this is enough to tell the two apart:

```python
import requests

def probe(base):
    # CKAN exposes an action API; Socrata exposes /api/views
    for name, path in [("ckan", "/api/3/action/site_read"),
                       ("socrata", "/api/views.json?limit=1")]:
        r = requests.get(base + path, timeout=15)
        print(name, r.status_code)
```

A 200 on one and a 404 on the other tells me which API I'm dealing with. A 401 or 403 tells me credentials are involved, which is a conversation to have early.

Once I knew the platform, I picked the integration approach per source instead of assuming one pattern would cover them all.

## Takeaways

- **Don't assume the portal platform.** "Open data portal" is a category, not a product. Identify it first.
- **Test a real call during discovery.** It's cheap and it surfaces auth requirements before they become a schedule problem.
- **Plan for more than one integration pattern.** If a project pulls from several public sources, expect them to differ.
- **Ask about credentials early.** Anything that needs auth needs an owner and a safe place to store the secret.
