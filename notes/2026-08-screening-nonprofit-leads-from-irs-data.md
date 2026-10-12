# Screening nonprofit leads from IRS bulk data

*Worked on: August 2026*

I wanted a repeatable way to find nonprofits that might be a good fit for GIS and data work, without starting every search by Googling organizations one at a time. Manual research is the expensive step, so the goal was to shrink the list as far as possible with data before any person looked at a website.

## The funnel

The starting point is the IRS Business Master File, the bulk list of exempt organizations, joined with Form 990 data for financials. From there I narrow in stages, cheapest filter first:

1. **Revenue band.** Organizations too small rarely have budget or staff for data projects. Very large ones usually have in-house teams or established vendors. I keep the middle band.
2. **Sector (NTEE code).** The National Taxonomy of Exempt Entities code tells you what the organization does. I keep the sectors where mapping and field data are a natural fit and drop the rest.
3. **Multi-chapter structure.** Organizations with chapters, affiliates, or regional branches tend to have the problem I solve: data spread across many places that needs to come together on one map. I flag those.
4. **Only then, manual research.** What's left is short enough to look at by hand.

## What was tricky

Bulk IRS files are wide, large, and not tidy. A few things cost me time:

- **Codes, not words.** NTEE and other fields are coded. I filter on code prefixes (the leading letter is the broad sector) rather than trying to match names.
- **Chapters are implicit.** There isn't a clean "has chapters" column. I look for signals such as shared names and group exemption relationships, so this flag is a hint to research, not a fact.
- **Order matters for effort.** Applying the sector filter before the revenue join meant carrying far more rows than necessary. Filtering on the cheap, local columns first keeps everything fast.

## A sketch of the filter

```python
import pandas as pd

bmf = pd.read_csv("bmf_extract.csv", dtype=str)
fin = pd.read_csv("form990_summary.csv", dtype=str)

df = bmf.merge(fin, on="EIN", how="inner")
df["REVENUE"] = pd.to_numeric(df["REVENUE"], errors="coerce")

leads = df[
    df["REVENUE"].between(MIN_REVENUE, MAX_REVENUE)
    & df["NTEE_CD"].str[:1].isin(TARGET_SECTORS)
]

# Flag likely multi-chapter organizations for a closer look
leads = leads.assign(multi_chapter=leads["GROUP_EXEMPTION"].fillna("0000") != "0000")
```

The thresholds and sector list live in constants at the top of the script so I can adjust the funnel without touching the logic.

## Takeaway

Lead research is a filtering problem before it's a research problem. Let public structured data do the first cuts (revenue, sector, structure), and spend human attention only on the short list that survives. Treat the chapter flag as a prompt to investigate, and keep the thresholds easy to change, because the right funnel shape only becomes clear after you've looked at a few results.
