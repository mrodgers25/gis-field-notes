# Fixing ArcGIS Hub sitemap XML

*Worked on: July 2026*

A community mapping site built on ArcGIS Hub wasn't showing up well in search results. The pages existed and the content was good, but search engines weren't indexing them the way they should. When I looked at the sitemap the site was publishing, I found three separate problems, and any one of them could have been enough to get the file ignored or only partly read.

## What went wrong

- **Unescaped ampersands.** Some page titles and URLs contained a raw `&`. In XML, an ampersand has to be written as `&amp;`. A bare one makes the whole document malformed, and a strict parser may reject the sitemap rather than skip the one bad entry.
- **Messy slugs.** Some page slugs were long, inconsistent, or carried leftovers from how the page was first created. They worked as links, but they were a poor fit for something a crawler is supposed to treat as the clean, canonical address of a page.
- **Duplicate pages.** The same content was reachable under more than one page, so the sitemap listed near-identical entries. That splits signals between URLs and wastes crawl attention.

## How I solved it

I started by treating the sitemap as data and checking it before changing anything. A quick parse shows immediately whether the XML is well formed:

```python
import xml.etree.ElementTree as ET

try:
    ET.parse("sitemap.xml")
    print("well-formed")
except ET.ParseError as e:
    print("broken:", e)   # points at the line/column of the bad character
```

The parse error pointed straight at the ampersands. From there:

1. **Escaped the ampersands** (`&` becomes `&amp;`) anywhere they appeared in the sitemap.
2. **Cleaned up the slugs** so each page had a short, readable, consistent address.
3. **Removed the duplicates**, keeping one page per piece of content so there was a single URL for each.

Then I re-ran the parse to confirm the file was valid before asking search engines to look at it again.

If you ever generate or post-process a sitemap in code, let a library do the escaping instead of hand-editing strings:

```python
from xml.sax.saxutils import escape

loc = escape("https://example.org/maps/parks-&-trails")
# https://example.org/maps/parks-&amp;-trails
```

## Takeaways

- **Validate the sitemap like any other data feed.** A well-formedness check takes seconds and catches the problem that silently breaks everything after it.
- **Escape, don't hope.** `&`, `<`, and `>` in text or URLs need escaping in XML. Use a library where you can.
- **Slugs are a design decision.** Choose them deliberately when pages are created; fixing them later means thinking about the old links too.
- **One page per piece of content.** Duplicates in a sitemap are easy to create in a site builder and easy to miss without looking at the actual file.
- **Check the plumbing, not just the pages.** The content was fine. What kept the site from being found was the file meant to help it get found.
