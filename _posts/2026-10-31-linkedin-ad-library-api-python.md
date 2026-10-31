---
title: "LinkedIn Ad Library API in Python: B2B competitor ad research"
description: "LinkedIn has no public API for its Ad Library. Here is a tested Python script that exports competitors' LinkedIn ads with copy, CTAs, landing pages, UTM tags and impressions to CSV, and summarises them per company."
actor: linkedin-ad-library-scraper
tags: python, linkedinads, marketing, b2b
---

If you sell to businesses, your competitors' LinkedIn ads tell you more than their website does: which offer they push this month (a demo, a webinar, a free audit), which CTA they use, which landing page and campaign name sits behind each ad, and, for ads shown in the EU, how many impressions it got and who it targeted.

All of that is public in the [LinkedIn Ad Library](https://www.linkedin.com/ad-library/). Getting it into a spreadsheet is another story:

1. **There is no Ad Library API.** LinkedIn's Marketing API covers your own ad accounts, not other advertisers.
2. **The details are one click deep.** The search page shows a thumbnail and a line of copy; headline, CTA, landing URL, impressions and targeting are on a separate detail page per ad.
3. **LinkedIn blocks scripts fast.** Datacenter IPs get blocked, so a do-it-yourself scraper becomes a residential-proxy project.

This tutorial uses the [LinkedIn Ad Library Scraper actor on Apify](https://apify.com/lagenuina_agency/linkedin-ad-library-scraper) as the API and builds a competitor report in one Python file: several companies in, one CSV and a per-company summary out.

## What you need

- Python 3.9+
- A free Apify account and your API token, from [Settings → Integrations](https://console.apify.com/settings/integrations)

```bash
export APIFY_TOKEN=your_token_here
pip install "apify-client>=1.7,<2"
```

No LinkedIn account or cookies are involved. The examples target the 1.x client, where `.call()` returns a plain dict.

## The short version: one function

```python
import os
from apify_client import ApifyClient

client = ApifyClient(os.environ["APIFY_TOKEN"])


def linkedin_ads(company, countries=(), limit=20):
    run = client.actor("lagenuina_agency/linkedin-ad-library-scraper").call(run_input={
        "companies": [company],
        "countries": list(countries),
        "resultsLimit": limit,
    })
    return client.dataset(run["defaultDatasetId"]).list_items().items


for ad in linkedin_ads("hubspot", countries=["DE"], limit=3):
    print(ad["adId"], ad["format"], ad["advertiserName"], "|", ad.get("headline"), "|", ad.get("impressions"))
```

Output, run on 31 October 2026:

```text
1592372173 SINGLE_IMAGE HubSpot | Bereit für den Wechsel? | 50k-100k
1592785023 SINGLE_IMAGE IXTENSA 🧡 HubSpot | Kostenloses HubSpot-Audit jetzt anfragen | 10k-20k
1585176223 SINGLE_IMAGE HubSpot | Bereit für den Wechsel? | 50k-100k
```

Two things to notice already. The second ad is not from HubSpot itself but from a HubSpot partner agency whose page name contains "HubSpot": `companies` uses LinkedIn's advertiser-name search, which matches similar names. And impressions are there because I asked for Germany: LinkedIn publishes impressions, countries and targeting only for ads shown in the EU.

## The full script: several competitors, one CSV, one summary

```python
"""B2B competitor research: LinkedIn ads of several companies to one CSV, plus a summary.

Usage: python competitors.py hubspot pipedrive salesforce --country DE --limit 50
"""
import argparse
import csv
import os
import sys
from collections import Counter, defaultdict

from apify_client import ApifyClient

ACTOR = "lagenuina_agency/linkedin-ad-library-scraper"
TOKEN = os.environ.get("APIFY_TOKEN")
if not TOKEN:
    sys.exit("Set APIFY_TOKEN first: https://console.apify.com/settings/integrations")

FIELDS = ["company", "adId", "format", "advertiserName", "paidBy", "headline", "body", "ctas",
          "landingDomain", "utm_campaign", "start", "end", "impressionsMin", "impressionsMax", "adUrl"]


def main():
    p = argparse.ArgumentParser()
    p.add_argument("companies", nargs="+", help="advertiser names as typed in the Ad Library")
    p.add_argument("--country", action="append", default=[], help="two-letter code, repeatable")
    p.add_argument("--period", choices=["last-30-days", "current-month", "current-year", "last-year"])
    p.add_argument("--limit", type=int, default=50, help="max ads per company")
    a = p.parse_args()

    run_input = {"companies": a.companies, "countries": a.country, "resultsLimit": a.limit}
    if a.period:
        run_input["dateOption"] = a.period
    client = ApifyClient(TOKEN)
    run = client.actor(ACTOR).call(run_input=run_input)
    ads = client.dataset(run["defaultDatasetId"]).list_items().items

    rows = []
    for ad in ads:
        # The run searches all companies at once; startUrl tells which search found the ad.
        company = next((c for c in a.companies if c.lower() in (ad.get("startUrl") or "").lower()), "")
        ran = ad.get("availability") or {}
        rows.append({**ad, "company": company, "ctas": " | ".join(ad.get("ctas") or []),
                     "utm_campaign": (ad.get("utm") or {}).get("utm_campaign"),
                     "start": ran.get("start"), "end": ran.get("end")})

    with open("linkedin_competitor_ads.csv", "w", newline="", encoding="utf-8") as f:
        w = csv.DictWriter(f, fieldnames=FIELDS, extrasaction="ignore")
        w.writeheader()
        w.writerows(rows)

    by_company = defaultdict(list)
    for r in rows:
        by_company[r["company"]].append(r)
    for company, items in by_company.items():
        print(f"\n== {company}: {len(items)} ads")
        print("  formats:  ", dict(Counter(r["format"] for r in items)))
        print("  CTAs:     ", dict(Counter(r["ctas"] or "-" for r in items)))
        print("  landing:  ", dict(Counter(r["landingDomain"] or "-" for r in items).most_common(3)))
        print("  paid by:  ", dict(Counter(r["paidBy"] or "-" for r in items).most_common(3)))
        top = max(items, key=lambda r: r.get("impressionsMax") or 0)
        print(f"  top reach: {top.get('impressionsMin')}-{top.get('impressionsMax')}  \"{top.get('headline')}\"")
    print(f"\n{len(rows)} ads -> linkedin_competitor_ads.csv")


if __name__ == "__main__":
    main()
```

How it works:

1. **One run for all companies.** `companies` takes a list and the actor runs one Ad Library search per name. `resultsLimit` caps each search, and with it the cost.
2. **`startUrl` tells you which search found each ad**, so the script can group rows by the competitor you asked for, even when the advertiser name differs (partners, regional pages).
3. **Flattens what a spreadsheet cannot hold**: CTAs become `a | b`, the `utm` dict gives its `utm_campaign`, `availability` becomes `start` and `end`.

The input fields used here:

| Field | Meaning |
|---|---|
| `companies` | Advertiser names, one search each |
| `keywords` | Words in the ad copy, e.g. `"crm"`, to search across all advertisers |
| `companyIds` | Numeric LinkedIn company IDs, for one exact advertiser |
| `countries` | Two-letter codes; empty means all countries |
| `dateOption` | `last-30-days`, `current-month`, `current-year` or `last-year` |
| `resultsLimit` | Max ads per company, keyword or URL |
| `skipDetails` | List data only: faster and cheaper, but no headline, CTA, landing page or impressions |

## Real output

Run on 31 October 2026, two CRM vendors, ads shown in Germany, five ads each:

```bash
python competitors.py hubspot pipedrive --country DE --limit 5
```

```text
== hubspot: 5 ads
  formats:   {'SINGLE_IMAGE': 5}
  CTAs:      {'Learn more': 2, 'Mehr erfahren': 3}
  landing:   {'hubs.la': 3, 'lp.ixtensa.de': 2}
  paid by:   {'HubSpot, Inc.': 3, 'IXTENSA GmbH & Co. KG': 2}
  top reach: 50000-100000  "Bereit für den Wechsel?"

== pipedrive: 5 ads
  formats:   {'JOB': 2, 'VIDEO': 3}
  CTAs:      {'-': 2, 'Learn more': 3}
  landing:   {'pipedrive.com': 3, '-': 2}
  paid by:   {'Pipedrive, Inc.': 3, '-': 2}
  top reach: 150000-200000  "%FIRSTNAME%, explore jobs at Pipedrive that match your skills"

10 ads -> linkedin_competitor_ads.csv
```

The run took about three and a half minutes: LinkedIn detail pages are slow to fetch, roughly 20 seconds per ad on this run. A few rows of the CSV (body shortened):

```csv
company,adId,format,advertiserName,paidBy,headline,body,ctas,landingDomain,utm_campaign,start,end,impressionsMin,impressionsMax,adUrl
hubspot,1592785023,SINGLE_IMAGE,IXTENSA 🧡 HubSpot,IXTENSA GmbH & Co. KG,Kostenloses HubSpot-Audit jetzt anfragen,Wenn die HubSpot-Maschine zum knirschenden Getriebe wird …,Learn more,lp.ixtensa.de,HubSpot-Audit,2026-10-27,2026-10-30,10000,20000,https://www.linkedin.com/ad-library/detail/1592785023
hubspot,1592372163,SINGLE_IMAGE,HubSpot,"HubSpot, Inc.",Bereit für den Wechsel?,"Deine Systeme kosten Zeit und Nerven? Diese Live-Demo ist für Teams, die über einen Wechsel zu HubSpot nachdenken.",Mehr erfahren,hubs.la,EMEA,2026-10-20,2026-10-31,50000,100000,https://www.linkedin.com/ad-library/detail/1592372163
pipedrive,915417333,VIDEO,Pipedrive,"Pipedrive, Inc.",Disco or Demo? 🪩🖥️,"Which should come first: the discovery call or the product demo? ...",Learn more,pipedrive.com,archetype-30mpc,2025-11-04,2025-11-11,1000,5000,https://www.linkedin.com/ad-library/detail/915417333
```

Even on ten ads the positioning is readable. HubSpot's own German ads push a single message, a live demo for teams thinking about switching ("Bereit für den Wechsel?"), with a German CTA, a `hubs.la` short link and an `EMEA` campaign tag, and reached 50k to 100k impressions. Pipedrive's three video ads all belong to one campaign built with a sales podcast (`utm_campaign` `archetype-30mpc`) that ended in November 2025, and its two ads still running are recruiting ads (format `JOB`, no CTA or landing page). Add `--period last-30-days` when you only care about what is live now. The `paidBy` column also shows a partner agency running HubSpot-branded audit offers on its own budget.

Each item has more than the CSV keeps: `imageUrls`, `videoUrl`, `document` for carousel/document ads, `impressionsPerCountry` with a `percent` per country, and `targeting` (for example `{"language": "English", "location": "Germany"}`) plus `targetingParameters`, which says whether company, job or audience targeting was used or excluded.

## Variants

**One-click examples.** Prefilled public tasks you can run from the Apify console: [HubSpot's LinkedIn ads](https://apify.com/lagenuina_agency/linkedin-ad-library-scraper/examples/hubspot-linkedin-ads), [LinkedIn ads by keyword](https://apify.com/lagenuina_agency/linkedin-ad-library-scraper/examples/linkedin-ads-by-keyword) and [LinkedIn ads about AI agents](https://apify.com/lagenuina_agency/linkedin-ad-library-scraper/examples/ai-agents-linkedin-ads).

**A whole category, not one company.** Replace `companies` with `keywords` (`"crm"`, `"cybersecurity"`) to see how every advertiser in a niche talks about it, then count CTAs and offers the same way.

**New ads only.** Add `"onlyNewAds": true` and a `monitorStoreName` per competitor, schedule the run daily, and later runs return (and bill) only ads you have not seen.

**JavaScript and n8n.** The example repo [assa83/linkedin-ad-library-api](https://github.com/assa83/linkedin-ad-library-api) has a one-company CSV export in Python and Node.js, and an n8n workflow.

**AI agents (MCP).** Give the actor to Claude, Cursor or any MCP client:

```json
{
  "mcpServers": {
    "linkedin-ad-library-scraper": {
      "url": "https://mcp.apify.com/?actors=lagenuina_agency/linkedin-ad-library-scraper"
    }
  }
}
```

Then ask "what LinkedIn ads did HubSpot run in Germany this month, and which agency paid for them?"

## What it costs

Pay per ad, no start fee. Ads with full details cost $3.20 per 1,000 on the Apify Free plan, down to $0.90 on Gold and above; list-only ads (`skipDetails`) cost $1.60 down to $0.44. The 13 detailed ads in my two test runs come to about four cents on the Free plan. Check the [actor page](https://apify.com/lagenuina_agency/linkedin-ad-library-scraper) for current prices.

## Limits worth knowing

- **Impressions, countries and targeting only for EU ads.** LinkedIn publishes them only for ads shown in the European Union; for other ads those fields are empty. Filter on an EU country if you need reach.
- **No spend, budget, clicks or CTR.** LinkedIn does not publish them.
- **Company names are fuzzy.** `companies` uses LinkedIn's advertiser-name search, so partners and similarly named pages show up (as in the run above). Use `companyIds` for one exact advertiser.
- **About one year of history**, as far back as the Ad Library goes.
- **Slow by nature.** Detail pages need a residential proxy and LinkedIn blocks aggressively; expect minutes, not seconds, for a few dozen ads, and an occasional search that has to be retried. Use `skipDetails` for a quick first look.
- **Job ads and some formats have no CTA or landing page**, so expect empty cells.

## Wrap-up

LinkedIn will not give you an Ad Library API, but its Ad Library is public. With a hosted scraper and one Python file you get competitors' offers, CTAs, landing pages, campaign names and EU reach in a CSV in a few minutes. Copy the script, set `APIFY_TOKEN`, list your competitors and pick an EU country if you want impressions.

---

*Disclosure: I built this actor. It is published on Apify by Quelli di Carne Genuina, and I earn from its usage.*
