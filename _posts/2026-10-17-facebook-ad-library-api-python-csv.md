---
title: "Facebook Ad Library API alternative: export competitor ads to CSV with Python"
description: "Meta's official Ad Library API only covers political and issue ads. Here is a tested Python script that exports any page's Facebook and Instagram ads to CSV, no developer app or ID verification."
actor: meta-ad-library-scraper
tags: python, facebookads, marketing, webscraping
---

Anyone who does paid social sooner or later wants the same thing: "show me every ad this competitor is running on Facebook and Instagram, in a spreadsheet."

The [Meta Ad Library](https://www.facebook.com/ads/library/) has the data, publicly. Getting it out is the hard part:

1. **The official Ad Library API is narrow.** It returns political and issue ads (plus ads delivered in the EU), and you need a Meta developer app and identity verification before the first call. Ordinary commercial ads from a brand outside the EU, the ones most marketers care about, are not there.
2. **The website is not an export tool.** You scroll, ads load ten at a time, and there is no "download CSV" button. Copying copy, CTAs and landing pages by hand for a hundred ads is an afternoon gone.
3. **Scraping it yourself means fighting rate limits.** Facebook rate-limits datacenter IPs on the Ad Library, so a quick script turns into a proxy project.

This tutorial uses the [Meta Ad Library Scraper actor on Apify](https://apify.com/lagenuina_agency/meta-ad-library-scraper) as the API and keeps the Python to one short file that writes a CSV you can open in Excel or Google Sheets.

## What you need

- Python 3.9+
- A free Apify account and your API token, from [Settings → Integrations](https://console.apify.com/settings/integrations)

```bash
export APIFY_TOKEN=your_token_here
pip install "apify-client>=1.7,<2"
```

No Facebook account, cookies or Meta developer app are involved.

## Find the page ID

The scraper accepts a numeric page ID, an `@username` or a facebook.com page link. To get the ID, open the brand in the Ad Library: it is the `view_all_page_id=` number in the URL. Nike's is `15087023444`. If you pass a plain brand name instead, it is searched as a keyword across all advertisers.

## The script

This is `python/example.py` from the example repo [assa83/facebook-ad-library-api](https://github.com/assa83/facebook-ad-library-api). I ran it today, unchanged.

```python
"""Export the Facebook & Instagram ads of a page (or a keyword) from the Meta Ad Library to CSV.

Usage: python example.py 15087023444 --country US --active
       python example.py "running shoes" --keyword --country US
"""
import argparse
import csv
import os
import sys

from apify_client import ApifyClient

ACTOR = "lagenuina_agency/meta-ad-library-scraper"
TOKEN = os.environ.get("APIFY_TOKEN")
if not TOKEN:
    sys.exit("Set APIFY_TOKEN first: https://console.apify.com/settings/integrations")

FIELDS = ["ad_archive_id", "page_name", "is_active", "start_date_iso", "end_date_iso", "publisher_platform",
          "display_format", "ad_text", "title", "cta_text", "link_url", "image_urls", "video_urls", "spend", "ad_library_url"]


def main():
    p = argparse.ArgumentParser()
    p.add_argument("target", help="page id, @username, facebook.com link, or a keyword with --keyword")
    p.add_argument("--keyword", action="store_true", help="search the target as a keyword across all advertisers")
    p.add_argument("--country", default="ALL", help="two-letter code or ALL")
    p.add_argument("--active", action="store_true", help="only ads running now")
    p.add_argument("--limit", type=int, default=200)
    a = p.parse_args()

    run_input = {"country": a.country, "activeStatus": "active" if a.active else "all", "maxAds": a.limit}
    run_input["keywords" if a.keyword else "pageIDs"] = [a.target]
    client = ApifyClient(TOKEN)
    run = client.actor(ACTOR).call(run_input=run_input)

    safe = "".join(c if c.isalnum() else "_" for c in a.target)[:40]
    out = f"meta_ads_{safe}.csv"
    n = 0
    with open(out, "w", newline="", encoding="utf-8") as f:
        w = csv.DictWriter(f, fieldnames=FIELDS, extrasaction="ignore")
        w.writeheader()
        for ad in client.dataset(run["defaultDatasetId"]).iterate_items():
            row = {k: (" | ".join(v) if isinstance(v, list) else v) for k, v in ad.items() if k in FIELDS}
            w.writerow(row)
            n += 1
    print(f"{n} ads -> {out}")


if __name__ == "__main__":
    main()
```

How it works:

1. **Builds the input.** `pageIDs` for one advertiser, or `keywords` to search every advertiser like the Ad Library search box. `activeStatus: "active"` matches what the website shows by default; `"all"` is the full archive, often ten times larger.
2. **`maxAds` caps the run**, and with it the cost. Leave it high for a full export.
3. **Flattens lists** (platforms, image and video URLs) into `a | b` strings so each ad is one CSV row.

## Real output

Run on 17 October 2026, Nike's active US ads, capped at 10:

```bash
python example.py 15087023444 --country US --active --limit 10
```

```text
INFO  Scraping 1 target(s), country=US, status=active, type=all, media=all, max 10 ads
INFO  Done: 10 ad(s). Ad Library pages=1, GraphQL pages=0, proxy rotations=0.
Status: SUCCEEDED
10 ads -> meta_ads_15087023444.csv
```

The run took about 6 seconds. A few columns of `meta_ads_15087023444.csv` (image URLs shortened):

```csv
ad_archive_id,page_name,is_active,start_date_iso,publisher_platform,display_format,ad_text,cta_text,link_url
1067080952624010,Nike,True,2026-09-01T07:00:00.000Z,FACEBOOK | INSTAGRAM,DPA,Ball’s in your court. Get the gear that never misses.,Shop now,https://www.nike.com/
968300552951603,Nike,True,2026-09-01T07:00:00.000Z,FACEBOOK | INSTAGRAM,DPA,Get the gear not afraid to put in the work.,Shop now,https://www.nike.com/
2816322322094811,Nike,True,2026-10-15T07:00:00.000Z,FACEBOOK | INSTAGRAM,DPA,Run through summer with gear that can take the heat.,Shop now,https://www.nike.com/
```

Every row also has `image_urls` (full-size creative on Facebook's CDN), `video_urls`, `title`, `end_date_iso` and an `ad_library_url` you can open to check the ad by hand.

## From CSV to insight in ten lines

A CSV is only useful if it answers a question. This snippet, run on the file above, shows when the ads started, which formats the brand uses and which messages it repeats:

```python
import csv
from collections import Counter

ads = list(csv.DictReader(open("meta_ads_15087023444.csv", encoding="utf-8")))
print(len(ads), "ads")
print("by start month:", Counter(a["start_date_iso"][:7] for a in ads).most_common())
print("by format:", Counter(a["display_format"] for a in ads).most_common())
print("top messages:")
for text, n in Counter(a["ad_text"] for a in ads).most_common(3):
    print(f"  {n}x  {text}")
```

```text
10 ads
by start month: [('2026-09', 6), ('2026-10', 3), ('2025-06', 1)]
by format: [('DPA', 10)]
top messages:
  3x  Get the gear that goes hard on and off the field.
  3x  Ball’s in your court. Get the gear that never misses.
  3x  Get the gear not afraid to put in the work.
```

Even on ten ads the pattern is visible: Nike's US catalog ads rotate three short lines of copy across many creatives, and a new batch went live in mid-October. On a full export (leave out `--active` to get the whole archive, and raise `--limit`) the same counts tell you how often a competitor refreshes creative and which angles survive for months.

## Variants

**Search a keyword instead of a page.** `python example.py "protein bars" --keyword --country US --active --limit 5` returns ads from every advertiser that mention the phrase. In my run the first five were all Nature Valley catalog ads, which is itself a finding.

**n8n, every Monday.** The repo includes an [n8n workflow](https://github.com/assa83/facebook-ad-library-api/tree/main/n8n) that runs the actor on a competitor's page each Monday at 08:00, flattens the fields and appends them to Google Sheets. Import the JSON, add a Header Auth credential (`Authorization: Bearer <token>`), set your page IDs and sheet.

**AI agents (MCP).** Give the actor to Claude, Cursor or any MCP client:

```json
{
  "mcpServers": {
    "meta-ad-library-scraper": {
      "url": "https://mcp.apify.com/?actors=lagenuina_agency/meta-ad-library-scraper"
    }
  }
}
```

Then ask "what are Nike's active Instagram ads in the US promoting this week?" and the agent runs the export for you.

## What it costs

$0.20 per 1,000 ads on every Apify plan, residential proxy included, no start fee. The 15 ads in my two test runs cost a fraction of a cent. Set `maxAds` or a maximum charge per run to cap spending. Check the [actor page](https://apify.com/lagenuina_agency/meta-ad-library-scraper) for current prices.

## Limits worth knowing

- **No spend or reach for commercial ads.** Meta publishes spend, impressions and reach only for political and issue ads, so those columns are empty for brands like Nike. That is a Meta limit, not a scraper one.
- **Catalog ads show templates.** Dynamic product ads (`DPA`, `DCO`) store copy as placeholders such as `{{product.name}}`; the Ad Library does not render the products. You still get the images, CTA and landing page.
- **Large archives take time.** Facebook serves ten ads per request, roughly six minutes for 1,000 ads of one page.
- **Media links expire.** Image and video URLs are signed CDN links valid for some days. Download what you want to keep.
- **Brand names are keywords.** A plain name like "nike" searches ad text across all advertisers. Use the page ID for one exact advertiser.
- The data is public transparency data, but you are responsible for how you use it. Do not process personal data without a legal basis.

## Wrap-up

If the official Ad Library API does not cover the ads you need, or the identity verification is a blocker, a hosted scraper plus forty lines of Python gets a competitor's Facebook and Instagram ads into a spreadsheet in seconds. Clone the [example repo](https://github.com/assa83/facebook-ad-library-api), set `APIFY_TOKEN` and point it at the page ID you care about.

---

*Disclosure: I built this actor. It is published on Apify by Quelli di Carne Genuina, and I earn from its usage.*
