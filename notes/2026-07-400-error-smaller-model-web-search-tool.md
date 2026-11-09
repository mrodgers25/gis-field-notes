# A 400 error from a smaller model and the web_search tool

*Worked on: July 2026*

I had a working pipeline that used an LLM with web search to fill gaps in a point layer. It ran fine on a larger model. When I pointed the same script at Haiku 4.5 to cut the cost of a bulk run, every request came back with a 400 error.

## What was tricky

The request body hadn't changed, and neither had my prompts or the data. The only difference was the model name, so my first instinct was to suspect the prompt or a malformed record. That was the wrong place to look.

The cause was the newer version of the web_search tool. It allows programmatic tool calling by default, which means the tool can be invoked from inside code execution rather than only directly by the model. The larger model accepted that. Haiku 4.5 rejected it outright, and the API answered with a 400 instead of quietly falling back.

## The fix

I told the tool to accept only direct calls by setting `allowed_callers` explicitly:

```python
tools = [{
    "type": "web_search_<newer_version>",   # the newer tool version
    "name": "web_search",
    "max_uses": 3,
    "allowed_callers": ["direct"],          # opt out of programmatic calling
}]

response = client.messages.create(
    model=MODEL,
    max_tokens=1024,
    tools=tools,
    messages=messages,
)
```

With `allowed_callers=["direct"]`, the smaller model accepted the request and the lookups worked again.

## The second problem: cached failures

My pipeline caches results per record so I never pay to look the same thing up twice. The failed run had already written its errors into that cache as "not found." So after fixing the tool config, a normal rerun skipped those records, because as far as the cache was concerned they'd been answered.

I had to rerun once with `--force` to bypass the cache and retry them. That's the same trap I wrote about in my note on failed API calls cached as "not found": a transient or configuration error should never be stored as if it were a real negative result.

## Takeaways

- A 400 that appears only when you swap the model is probably a capability mismatch in the tool config, not your prompt or your data. Check the tool options first.
- Newer tool versions can change defaults. Pinning behavior explicitly, like `allowed_callers=["direct"]`, makes the request work across models instead of depending on whatever the default happens to be.
- Don't cache errors as answers. Store failures separately, or don't store them, so a config fix doesn't need a manual `--force` to undo the damage.
- Keep a force flag in any cached pipeline. I was glad it existed.
