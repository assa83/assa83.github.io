---
title: "Google Trends API in Python: a working pytrends alternative (2026)"
description: "pytrends is archived and 429 errors are common. Here is a small, tested Python script that exports Google Trends interest over time and rising queries to CSV."
actor: google-trends-scraper
tags: python, googletrends, webscraping, api
---

Google has never shipped a general-purpose public API for Google Trends. For years the Python answer was [pytrends](https://github.com/GeneralMills/pytrends), an unofficial library that called the same endpoints the Trends website uses.

Two things changed that:

1. **The pytrends repository is archived on GitHub.** It is read-only, so no new fixes will land when Google changes something on its side.
2. **`429 Too Many Requests` is the error everyone meets.** Trends throttles automated traffic hard. A few requests from one IP and you start getting 429s, and then you are writing retry loops, sleeping between calls and rotating proxies instead of analysing data.

If you just need the numbers (interest over time, rising queries, interest by region), a simpler option is to let a hosted scraper deal with sessions and rate limits and call it from Python. This tutorial does that with the [Google Trends Scraper actor on Apify](https://apify.com/lagenuina_agency/google-trends-scraper) and the official `apify-client` package.

By the end you will have a script that writes two CSV files you can open in Excel or import into Google Sheets:

- `trends_YYYY-MM-DD.csv`: one row per keyword and date (`searchTerm, geo, date, value, isPartial`)
- `rising_YYYY-MM-DD.csv`: the rising related queries for each keyword (`searchTerm, query, growth`)

The full code is in the repo [assa83/google-trends-api-python](https://github.com/assa83/google-trends-api-python), with JavaScript and n8n versions too.

## What you need

- Python 3.9+
- A free Apify account and your API token, from [Settings → Integrations](https://console.apify.com/settings/integrations)

Keep the token out of your code. Put it in an environment variable:

```bash
export APIFY_TOKEN=your_token_here
pip install "apify-client>=1.7,<2"
```

The example targets the 1.x client, where `.call()` returns a plain dict. If you use apify-client 2.x, check the client changelog for the return type.

## The script

Save this as `example.py`. It is the same file as `python/example.py` in the repo, and I ran it today against the live actor.

```python
"""Weekly Google Trends export to CSV (Google Sheets ready).

Usage: python example.py "chatgpt" "gemini" --geo US
"""
import argparse
import csv
import datetime
import os
import sys

from apify_client import ApifyClient

ACTOR = "lagenuina_agency/google-trends-scraper"
TOKEN = os.environ.get("APIFY_TOKEN")
if not TOKEN:
    sys.exit("Set APIFY_TOKEN first: https://console.apify.com/settings/integrations")


def main():
    p = argparse.ArgumentParser()
    p.add_argument("terms", nargs="+", help="keywords to track")
    p.add_argument("--geo", default="", help="country/region code, empty = worldwide")
    p.add_argument("--time-range", default="today 3-m")
    a = p.parse_args()

    client = ApifyClient(TOKEN)
    run = client.actor(ACTOR).call(run_input={
        "searchTerms": a.terms,
        "geo": a.geo,
        "timeRange": a.time_range,
        "includeInterestByRegion": False,
        "includeRelated": True,
    })
    items = client.dataset(run["defaultDatasetId"]).list_items().items

    today = datetime.date.today().isoformat()
    with open(f"trends_{today}.csv", "w", newline="", encoding="utf-8") as f:
        w = csv.writer(f)
        w.writerow(["searchTerm", "geo", "date", "value", "isPartial"])
        for it in items:
            for point in it.get("timeline", []):
                w.writerow([it["searchTerm"], it.get("geo", ""), point["date"][:10], point.get("value"), point.get("isPartial", False)])
    with open(f"rising_{today}.csv", "w", newline="", encoding="utf-8") as f:
        w = csv.writer(f)
        w.writerow(["searchTerm", "query", "growth"])
        for it in items:
            for q in it.get("relatedQueries_rising", []):
                w.writerow([it["searchTerm"], q["query"], q.get("formattedValue", q.get("value"))])
    print(f"{len(items)} keywords -> trends_{today}.csv, rising_{today}.csv")


if __name__ == "__main__":
    main()
```

What it does, step by step:

1. **Reads the token** from `APIFY_TOKEN` and stops with a clear message if it is missing.
2. **Starts the actor** with `client.actor(...).call(run_input=...)`. `call()` blocks until the run finishes, so there is no polling code to write.
3. **Reads the results** from the run's default dataset. Each item is one keyword with its `timeline`, related queries and metadata.
4. **Flattens** the nested JSON into two CSV files.

The input fields used here:

| Field | Meaning |
|---|---|
| `searchTerms` | List of keywords, one result per keyword |
| `geo` | Country or region code such as `US`, `IT`, `GB`; empty string means worldwide |
| `timeRange` | Same syntax as the Trends website, e.g. `today 3-m`, `today 12-m` |
| `includeInterestByRegion` | Turned off here to keep the output small |
| `includeRelated` | Adds top and rising related queries |

The [actor page](https://apify.com/lagenuina_agency/google-trends-scraper) (Input tab) documents the rest, including a Trending Now mode.

## Running it

```bash
python example.py "chatgpt" "claude"
```

The run takes a few seconds per keyword. The console log ends like this:

```text
INFO  Scraping 2 keyword job(s)
INFO  Done: chatgpt
INFO  Done: claude
Status: SUCCEEDED, Message: 2 result(s)
2 keywords -> trends_2026-10-17.csv, rising_2026-10-17.csv
```

## Real output

This is an excerpt from a run on 17 October 2026, worldwide, over a one-month window. Over a short window Trends returns daily points. `trends_2026-10-17.csv`:

```csv
searchTerm,geo,date,value,isPartial
chatgpt,Worldwide,2026-09-17,74,False
chatgpt,Worldwide,2026-09-18,64,False
chatgpt,Worldwide,2026-09-19,55,False
...
chatgpt,Worldwide,2026-10-15,99,False
chatgpt,Worldwide,2026-10-16,84,False
chatgpt,Worldwide,2026-10-17,74,True
claude,Worldwide,2026-09-17,66,False
claude,Worldwide,2026-09-18,58,False
...
claude,Worldwide,2026-10-16,90,False
claude,Worldwide,2026-10-17,73,True
```

And `rising_2026-10-17.csv`:

```csv
searchTerm,query,growth
chatgpt,chatgpt dots,"+1,200%"
claude,opus 5.5,Breakout
claude,claude opus 5.5,Breakout
claude,claude startup program,"+2,650%"
claude,claude for startups,+750%
claude,claude mods,+650%
```

Some notes on reading this data:

- **Values are relative, not search volumes.** Like the Trends website, 100 is the peak inside the requested keyword set, time range and region. Change any of those and the numbers are rescaled.
- **`isPartial: True`** marks the latest point, which is still being collected. Drop it before you compare it with earlier points, or it will look like a sudden fall.
- **"Breakout"** in rising queries means growth over 5,000%, so Trends shows a label instead of a percentage.

If you want the raw JSON rather than CSV, each dataset item looks like this (shortened, from a run for `chatgpt` in Italy over 12 months):

```json
{
  "type": "keyword",
  "searchTerm": "chatgpt",
  "geo": "IT",
  "timeRange": "today 12-m",
  "data_granularity": "week",
  "timeline": [
    { "date": "2026-10-11T00:00:00.000Z", "value": 88, "isPartial": true }
  ],
  "relatedQueries_top": [
    { "query": "chatgpt gratis", "value": 100, "formattedValue": "100" }
  ],
  "relatedQueries_rising": [
    { "query": "chatgpt atlas", "value": 4250, "formattedValue": "+4,250%" }
  ],
  "trendsUrl": "https://trends.google.com/trends/explore?q=chatgpt&date=today+12-m&hl=en-US&geo=IT"
}
```

The `trendsUrl` field is useful for spot checks: open it and compare the chart with your CSV.

## Loading it into pandas

The CSV is already in long format, so pivoting it for a chart is short:

```python
import pandas as pd

df = pd.read_csv("trends_2026-10-17.csv", parse_dates=["date"])
df = df[~df["isPartial"]]  # drop the incomplete last point
wide = df.pivot(index="date", columns="searchTerm", values="value")
print(wide.tail())
```

## Scheduling it

The script names its files by date, so running it on a schedule builds a history without overwriting anything:

- **cron** on your own machine: `0 8 * * 1 cd /path/to/script && python example.py "brand a" "brand b" --geo US`
- **Apify Schedules**: schedule the actor itself in the Apify Console and read the dataset later.
- **n8n**: the repo includes a workflow that appends the rows to Google Sheets every Monday. I wrote a separate post on setting it up.

## What it costs

The actor uses pay-per-result pricing with no start fee. One keyword with all its data counts as one result. The price per 1,000 results depends on your Apify plan: **$2.40 on the Free plan, going down to $0.24 on Gold**. Tracking 10 keywords every week is about 520 results a year. Check the [actor page](https://apify.com/lagenuina_agency/google-trends-scraper) for current prices before you budget.

## Using it from AI agents (MCP)

Apify runs an MCP server, so you can give the actor to Claude, Cursor, VS Code or any other MCP client as a tool, without writing Python:

```json
{
  "mcpServers": {
    "google-trends-scraper": {
      "url": "https://mcp.apify.com/?actors=lagenuina_agency/google-trends-scraper"
    }
  }
}
```

Sign in with Apify OAuth when the client asks, or send `Authorization: Bearer <APIFY_TOKEN>`. Then you can ask questions such as "compare interest in X and Y in Germany over the last 12 months and list the rising queries" and the agent calls the actor for you.

## Limits worth knowing

- This is still Google Trends data, with the same sampling and normalisation as the website. Two runs for the same query can differ slightly.
- In the run above each keyword came back as its own job, with its own 0–100 scale (both peak near 99). So `chatgpt 74` next to `claude 73` does not mean equal search volume. Compare shapes and changes over time, not absolute values across keywords.
- The scraper reads only publicly available data. It is not affiliated with or endorsed by Google. Use it in line with the applicable terms and laws.

## Wrap-up

If pytrends stopped working for you, or you are tired of handling 429s, moving the scraping to a hosted actor and keeping your Python to about 40 lines of CSV code is a practical trade-off. Clone the [example repo](https://github.com/assa83/google-trends-api-python), set `APIFY_TOKEN` and run it.

---

*Disclosure: I built this actor. It is published on Apify by Quelli di Carne Genuina, and I earn from its usage.*
