---
title: "Google Ads Transparency Center API in Python: track competitor ads weekly"
description: "Google has no public API for the Ads Transparency Center. Here is a tested Python script that pulls a competitor's Google Search, YouTube and Display ads and appends only the new ones to a CSV every week."
actor: google-ads-transparency-scraper
tags: python, googleads, marketing, webscraping
---

Every ad that runs on Google Search, YouTube, Display, Shopping, Maps and Play ends up in the [Google Ads Transparency Center](https://adstransparency.google.com/). Type a competitor's domain and you see their creatives, when each one was first and last shown, and in which countries.

That is exactly the data you want for competitor research. The problem is getting it out:

1. **There is no official API.** Google publishes the Transparency Center as a website only. The Google Ads API covers your own account, not other advertisers.
2. **The website is built for looking, not for tracking.** There is no export, no "what's new since last week" view, and the page loads ads in batches as you scroll.
3. **Watching a competitor means doing it again every week**, and remembering which creatives you had already seen.

This tutorial uses the [Google Ads Transparency Scraper actor on Apify](https://apify.com/lagenuina_agency/google-ads-transparency-scraper) as the API and builds a weekly monitor in one short Python file: each run appends only the competitor's new ads to a CSV.

## What you need

- Python 3.9+
- A free Apify account and your API token, from [Settings → Integrations](https://console.apify.com/settings/integrations)

```bash
export APIFY_TOKEN=your_token_here
pip install "apify-client>=1.7,<2"
```

No Google account, cookies or Google Ads access are involved. The examples target the 1.x client, where `.call()` returns a plain dict.

## The short version: one function

If you just want a competitor's ads as a list of dicts:

```python
import os
from apify_client import ApifyClient

client = ApifyClient(os.environ["APIFY_TOKEN"])


def google_ads(domain, region="US", max_ads=20):
    run = client.actor("lagenuina_agency/google-ads-transparency-scraper").call(
        run_input={"queries": [domain], "region": region, "maxAdsPerSearch": max_ads}
    )
    return client.dataset(run["defaultDatasetId"]).list_items().items


for ad in google_ads("hellofresh.com", max_ads=5):
    print(ad["firstShown"][:10], ad["lastShown"][:10], ad["shownForDays"], ad["format"], ad["advertiserName"])
```

Output, run on 31 October 2026:

```text
2026-03-16 2026-10-31 230 TEXT Grocery Delivery E-Services USA Inc.
2026-05-13 2026-10-31 171 IMAGE Grocery Delivery E-Services USA Inc.
2026-03-17 2026-10-31 225 TEXT Grocery Delivery E-Services USA Inc.
2025-09-29 2026-10-31 389 TEXT Grocery Delivery E-Services USA Inc.
2023-06-22 2026-10-31 1079 IMAGE Grocery Delivery E-Services USA Inc.
```

`queries` accepts a domain (`hellofresh.com`) or a brand name (`HelloFresh`). Note the advertiser name: the ads for hellofresh.com are paid by a legal entity with a different name, which is why searching by domain is usually the safer choice.

## The weekly monitor: only new ads, appended to a CSV

The actor has a monitoring mode: with `onlyNewAds: true` it remembers the creative IDs it has already returned (in a named key-value store on your Apify account) and outputs only ads it has not seen before. The script below gives each competitor and country its own memory, then appends the new ads to a CSV with the date they were found.

```python
"""Weekly Google Ads monitor: append a competitor's new ads to a CSV.

Usage: python monitor.py hellofresh.com --region US
       python monitor.py "Gymshark" --region GB --max 300
"""
import argparse
import csv
import datetime
import os
import re
import sys
from collections import Counter

from apify_client import ApifyClient

ACTOR = "lagenuina_agency/google-ads-transparency-scraper"
TOKEN = os.environ.get("APIFY_TOKEN")
if not TOKEN:
    sys.exit("Set APIFY_TOKEN first: https://console.apify.com/settings/integrations")

FIELDS = ["found_on", "creativeId", "advertiserName", "domain", "format", "firstShown", "lastShown",
          "shownForDays", "imageUrl", "previewUrl", "adUrl"]


def main():
    p = argparse.ArgumentParser()
    p.add_argument("competitor", help="domain (hellofresh.com), brand name or AR... advertiser ID")
    p.add_argument("--region", default="US", help="two-letter country code or ANYWHERE")
    p.add_argument("--max", type=int, default=200, help="max ads per advertiser")
    a = p.parse_args()

    slug = re.sub(r"[^a-z0-9-]+", "-", a.competitor.lower()).strip("-")
    target = {"advertiserIds": [a.competitor]} if a.competitor.startswith("AR") else {"queries": [a.competitor]}
    client = ApifyClient(TOKEN)
    run = client.actor(ACTOR).call(run_input={
        **target,
        "region": a.region,
        "maxAdsPerSearch": a.max,
        "onlyNewAds": True,                                  # return (and bill) only unseen ads
        "monitorStoreName": f"gads-{slug}-{a.region.lower()}",  # one memory per competitor + country
    })
    ads = client.dataset(run["defaultDatasetId"]).list_items().items

    out = f"google_ads_{slug}_{a.region.lower()}.csv"
    new_file = not os.path.exists(out)
    today = datetime.date.today().isoformat()
    with open(out, "a", newline="", encoding="utf-8") as f:
        w = csv.DictWriter(f, fieldnames=FIELDS, extrasaction="ignore")
        if new_file:
            w.writeheader()
        for ad in ads:
            w.writerow({**ad, "found_on": today})

    print(f"{len(ads)} new ads for {a.competitor} ({a.region}) -> {out}")
    if ads:
        print("by format:", dict(Counter(ad["format"] for ad in ads)))
        for ad in sorted(ads, key=lambda x: x.get("firstShown") or "", reverse=True)[:5]:
            print(f"  first shown {ad['firstShown'][:10]}  {ad['format']:<5} {ad['adUrl']}")


if __name__ == "__main__":
    main()
```

The input fields used here:

| Field | Meaning |
|---|---|
| `queries` | Domains or brand names. Brand names are matched through Google's advertiser search |
| `advertiserIds` | Exact advertisers, the `AR…` ID from the advertiser page URL |
| `region` | Two-letter country code, or `ANYWHERE` |
| `maxAdsPerSearch` | Cap per domain or advertiser, and with it the cost |
| `onlyNewAds` + `monitorStoreName` | Remember creative IDs between runs and return only new ones |

Other filters you can add to `run_input`: `platforms` (`SEARCH`, `YOUTUBE`, `SHOPPING`, `MAPS`, `PLAY`), `adFormat` (`TEXT`, `IMAGE`, `VIDEO`), and `startDate` / `endDate` as `YYYY-MM-DD`.

## Real output

First run, 31 October 2026, capped at 10 ads to keep the test cheap:

```bash
python monitor.py hellofresh.com --region US --max 10
```

```text
10 new ads for hellofresh.com (US) -> google_ads_hellofresh-com_us.csv
by format: {'TEXT': 8, 'IMAGE': 2}
  first shown 2026-08-06  TEXT  https://adstransparency.google.com/advertiser/AR00930880903813529601/creative/CR06191140733789929473?region=US
  first shown 2026-05-13  IMAGE https://adstransparency.google.com/advertiser/AR00930880903813529601/creative/CR05381931924145045505?region=US
  first shown 2026-04-29  TEXT  https://adstransparency.google.com/advertiser/AR00930880903813529601/creative/CR05704963286424879105?region=US
  first shown 2026-03-17  TEXT  https://adstransparency.google.com/advertiser/AR00930880903813529601/creative/CR06021832573225795585?region=US
  first shown 2026-03-16  TEXT  https://adstransparency.google.com/advertiser/AR00930880903813529601/creative/CR12390107782756433921?region=US
```

The first run always returns everything it scans, because the memory is empty. Running the same command again a minute later:

```text
INFO  hellofresh.com: 10 ads scanned
INFO  Done: 0 result(s).
0 new ads for hellofresh.com (US) -> google_ads_hellofresh-com_us.csv
```

Ten ads scanned, zero returned, and zero charged: in monitoring mode you pay only for the ads that are new. Each run finished in under 5 seconds. The first rows of the CSV (preview URL shortened):

```csv
found_on,creativeId,advertiserName,domain,format,firstShown,lastShown,shownForDays,imageUrl,previewUrl,adUrl
2026-10-31,CR17624809799952629761,Grocery Delivery E-Services USA Inc.,hellofresh.com,TEXT,2026-03-16T21:13:04.000Z,2026-10-31T07:06:21.000Z,230,https://tpc.googlesyndication.com/archive/simgad/6735410367972731805,,https://adstransparency.google.com/advertiser/AR00930880903813529601/creative/CR17624809799952629761?region=US
2026-10-31,CR05381931924145045505,Grocery Delivery E-Services USA Inc.,hellofresh.com,IMAGE,2026-05-13T13:22:56.000Z,2026-10-31T07:00:26.000Z,171,,https://displayads-formats.googleusercontent.com/ads/preview/content.js?client=ads-integrity-transparency&...,https://adstransparency.google.com/advertiser/AR00930880903813529601/creative/CR05381931924145045505?region=US
```

Each row has either an `imageUrl` (a rendered image of the ad) or a `previewUrl` (Google's own preview script), plus `adUrl` to open the creative in the Transparency Center.

## Schedule it weekly

Any scheduler works, because the memory lives on Apify, not on your machine. A cron line on a server:

```bash
0 8 * * MON  cd /opt/ads-monitor && APIFY_TOKEN=... python monitor.py hellofresh.com --region US >> monitor.log
```

If you prefer not to run anything yourself, create an Apify Schedule on the actor with the same input (`onlyNewAds: true` and a `monitorStoreName`) and add an email, Slack or webhook integration.

## From CSV to insight in ten lines

`shownForDays` is the most useful column. Advertisers rarely keep paying for ads that do not work, so the creatives that have been running for months are the ones worth studying. This snippet, run on the file above:

```python
import csv
from collections import Counter

ads = list(csv.DictReader(open("google_ads_hellofresh-com_us.csv", encoding="utf-8")))
print(len(ads), "ads,", "formats:", dict(Counter(a["format"] for a in ads)))
print("started per year:", sorted(Counter(a["firstShown"][:4] for a in ads).items()))
print("longest running:")
for a in sorted(ads, key=lambda a: int(a["shownForDays"]), reverse=True)[:3]:
    print(f"  {a['shownForDays']:>5} days  {a['format']:<5} since {a['firstShown'][:10]}  {a['creativeId']}")
```

```text
10 ads, formats: {'TEXT': 8, 'IMAGE': 2}
started per year: [('2023', 1), ('2025', 2), ('2026', 7)]
longest running:
   1079 days  IMAGE since 2023-06-22  CR08275988769178386433
    389 days  TEXT  since 2025-09-29  CR10771736042307321857
    389 days  TEXT  since 2025-09-29  CR01976191126645768193
```

One image ad has been running since June 2023 and was still shown today. On a full export (raise `--max`) the same three lines tell you how often a competitor refreshes creative and which ads they never switch off.

## Variants

**One-click examples.** Prefilled public tasks you can run from the Apify console without writing code: [Nike's Google ads](https://apify.com/lagenuina_agency/google-ads-transparency-scraper/examples/nike-google-ads), [a competitor's ads in the UK](https://apify.com/lagenuina_agency/google-ads-transparency-scraper/examples/competitor-google-ads-uk), [Amazon's Google ads](https://apify.com/lagenuina_agency/google-ads-transparency-scraper/examples/amazon-google-ads) and [Temu's Google ads](https://apify.com/lagenuina_agency/google-ads-transparency-scraper/examples/temu-google-ads).

**Who advertises at all?** Set `"resultType": "advertisers"` and pass a list of companies: you get one row per advertiser with Google's approximate ad count range instead of individual ads. Handy for qualifying leads.

**JavaScript and n8n.** The example repo [assa83/google-ads-transparency-api](https://github.com/assa83/google-ads-transparency-api) has the same monitor in Node.js and an n8n workflow.

**AI agents (MCP).** Give the actor to Claude, Cursor or any MCP client:

```json
{
  "mcpServers": {
    "google-ads-transparency-scraper": {
      "url": "https://mcp.apify.com/?actors=lagenuina_agency/google-ads-transparency-scraper"
    }
  }
}
```

Then ask "which YouTube video ads did nike.com run in Italy this month?" and the agent fills in the domain, region, platform and dates itself.

## What it costs

Pay per ad, no start fee: $1.60 per 1,000 ads on the Apify Free plan, down to $1.00 on Gold and above. Full creative details (`includeDetails`, with every A/B variation and the list of countries where the ad ran) add $2.40 per 1,000 on Free, $1.50 on Gold, and only if you turn them on. In monitoring mode, ads you have already seen are not charged, so a weekly run on a competitor with no new creatives costs nothing for results. The 15 ads in my test runs come to about two and a half cents at Free-plan prices. Check the [actor page](https://apify.com/lagenuina_agency/google-ads-transparency-scraper) for current prices.

## Limits worth knowing

- **No spend, impressions or clicks.** The Transparency Center does not publish them for commercial ads, so no tool can give them to you.
- **No ad copy as plain text.** Text, video and rich ads come as Google-hosted previews (`imageUrl` or `previewUrl`), not as a headline and description string.
- **One region per run.** For several countries, run once per country; the script already keeps a separate memory per country.
- **Brand names can be ambiguous.** A brand-name query can match several advertiser accounts, or a similarly named one. Use the domain or the exact `AR…` advertiser ID.
- **The monitor only sees what it scans.** New ads are detected among the `--max` ads scanned per run. Keep it above the number of ads the competitor runs.
- **Platform filters** are applied by Google only to ads shown from September 2023.

## Wrap-up

There is no Google Ads Transparency Center API, but a hosted scraper and fifty lines of Python give you the next best thing: a CSV that grows by exactly the ads a competitor launched since last week. Copy the script, set `APIFY_TOKEN`, point it at a domain and schedule it.

---

*Disclosure: I built this actor. It is published on Apify by Quelli di Carne Genuina, and I earn from its usage.*
