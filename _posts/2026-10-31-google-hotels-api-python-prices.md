---
title: "Google Hotels API in Python: compare hotel prices across dates"
description: "Google does not offer a public Google Hotels API. Here is a tested Python script that gets hotel prices for any dates, with the rate on every booking site, and finds the cheapest dates for the hotels you care about."
actor: google-hotels-scraper
tags: python, travel, api, automation
---

Google Hotels is the quickest way to see what a hotel costs on specific dates: one search shows the nightly price on the hotel's own website, Booking.com, Expedia, Hotels.com, Agoda and a dozen smaller sites side by side.

If you want that data in code (a price tracker, a "which weekend is cheapest" tool, a rate-parity check for your own hotel), you hit a wall:

1. **There is no public Google Hotels API.** Google's Hotel APIs are for hotels and partners sending their own prices into Google, not for reading prices out.
2. **The page is a JavaScript app.** Prices load asynchronously and each booking site's rate sits behind another request, so a quick `requests` + BeautifulSoup script returns nothing useful.
3. **Dates multiply the work.** Comparing three weekends for five hotels is fifteen searches, every day if you are tracking.

This tutorial uses the [Google Hotels Scraper actor on Apify](https://apify.com/lagenuina_agency/google-hotels-scraper) as the API and keeps the Python to one file that answers a concrete question: for these hotels, which check-in date is cheapest, and on which site?

## What you need

- Python 3.9+
- A free Apify account and your API token, from [Settings → Integrations](https://console.apify.com/settings/integrations)

```bash
export APIFY_TOKEN=your_token_here
pip install "apify-client>=1.7,<2"
```

No Google account or proxies to manage. The examples target the 1.x client, where `.call()` returns a plain dict.

## The short version: one function

Prices for a city and a pair of dates:

```python
import os
from apify_client import ApifyClient

client = ApifyClient(os.environ["APIFY_TOKEN"])


def hotel_prices(where, check_in, check_out, currency="USD", max_hotels=20):
    run = client.actor("lagenuina_agency/google-hotels-scraper").call(run_input={
        "locations": [where],
        "checkInDate": check_in,
        "checkOutDate": check_out,
        "adults": 2,
        "currency": currency,
        "maxResults": max_hotels,
    })
    return client.dataset(run["defaultDatasetId"]).list_items().items


for h in hotel_prices("hotels in Lisbon", "2026-12-04", "2026-12-06", "EUR", max_hotels=3):
    print(f"{h['pricePerNight']:>7} {h['currency']}/night  {h['lowestPriceVendor']:<12} {h['overallRating']}  {h['hotelName']}")
```

Output, run on 31 October 2026:

```text
 138.07 EUR/night  goseek.com   4.6  Brown's | Avenue Hotel
 155.83 EUR/night  Vio.com      4.7  Lisboa Pessoa Hotel
 221.45 EUR/night  The One Palácio da Anunciada 4.7  The One Palácio da Anunciada
```

`locations` is searched exactly as you would type it in Google Hotels ("hotels in Lisbon", "Paris", "hotels near Times Square"). `lowestPriceVendor` is the site with the lowest nightly rate; in the last row it is the hotel's own website. Dates can also be relative, such as `"30 days"`, which is handy for scheduled runs.

## The full script: the same hotels across several dates

The actor takes one check-in and check-out per run, so to compare dates we start one run per date. `.start()` returns immediately, so all runs go in parallel and the whole comparison takes about as long as a single run.

```python
"""Compare Google Hotels prices for the same hotels across several dates, to CSV.

Usage: python compare_dates.py "Lumen Hotel Lisbon" "Brown's Avenue Hotel Lisbon" \
           --checkin 2026-12-04 2026-12-11 2026-12-18 --nights 2 --currency EUR
"""
import argparse
import csv
import datetime as dt
import os
import sys

from apify_client import ApifyClient

ACTOR = "lagenuina_agency/google-hotels-scraper"
TOKEN = os.environ.get("APIFY_TOKEN")
if not TOKEN:
    sys.exit("Set APIFY_TOKEN first: https://console.apify.com/settings/integrations")


def main():
    p = argparse.ArgumentParser()
    p.add_argument("hotels", nargs="+", help="hotel names, Google Hotels URLs or entity IDs")
    p.add_argument("--checkin", nargs="+", required=True, help="check-in dates, YYYY-MM-DD")
    p.add_argument("--nights", type=int, default=1)
    p.add_argument("--adults", type=int, default=2)
    p.add_argument("--currency", default="USD")
    a = p.parse_args()

    client = ApifyClient(TOKEN)
    # One run per date, all started at once: total time is about one run, not one per date.
    runs = {}
    for d in a.checkin:
        out = (dt.date.fromisoformat(d) + dt.timedelta(days=a.nights)).isoformat()
        runs[d] = client.actor(ACTOR).start(run_input={
            "hotels": a.hotels, "checkInDate": d, "checkOutDate": out,
            "adults": a.adults, "currency": a.currency, "includeBookingSources": True,
        })

    rows = []
    for d, run in runs.items():
        run = client.run(run["id"]).wait_for_finish()
        for h in client.dataset(run["defaultDatasetId"]).iterate_items():
            rows.append({
                "checkIn": d, "hotel": h.get("hotelName"), "pricePerNight": h.get("pricePerNight"),
                "total": h.get("totalRate"), "cheapestSite": h.get("lowestPriceVendor"),
                "officialSite": h.get("officialSitePrice"),
                "cheaperThanOfficial": " | ".join(h.get("cheaperThanOfficialSite") or []),
                "sites": len(h.get("bookingSources") or []), "url": h.get("googleHotelsUrl"),
            })

    out = "hotel_prices_by_date.csv"
    with open(out, "w", newline="", encoding="utf-8") as f:
        w = csv.DictWriter(f, fieldnames=list(rows[0]) if rows else ["checkIn"])
        w.writeheader()
        w.writerows(rows)

    # Price grid: one line per hotel, one column per check-in date, cheapest date marked with *.
    print(f"{'hotel':<34}" + "".join(f"{d:>13}" for d in a.checkin))
    for name in dict.fromkeys(r["hotel"] for r in rows):
        prices = {r["checkIn"]: r["pricePerNight"] for r in rows if r["hotel"] == name}
        best = min((v for v in prices.values() if v is not None), default=None)
        cells = [f"{prices.get(d) or '-'}{'*' if prices.get(d) == best else ''}" for d in a.checkin]
        print(f"{name[:33]:<34}" + "".join(f"{c:>13}" for c in cells))
    print(f"{len(rows)} rows -> {out} (prices per night, {a.currency})")


if __name__ == "__main__":
    main()
```

The input fields that matter:

| Field | Meaning |
|---|---|
| `locations` | A search as typed in Google Hotels; returns up to `maxResults` hotels |
| `hotels` | Specific properties: a name, a Google Hotels URL or an `entityId` from a previous run |
| `checkInDate` / `checkOutDate` | `YYYY-MM-DD`, or days from today such as `"30 days"` |
| `adults` / `childrenAges` | Occupancy of one room, e.g. `2` and `[5, 9]` |
| `currency`, `countryCode`, `languageCode` | Currency, Google market (`gl`) and language (`hl`) |
| `includeBookingSources` | The rate of every booking site, with links (on by default, no extra cost) |
| `minPrice`, `maxPrice`, `minRating`, `hotelClass` | Filters; hotels filtered out are not charged |

## Real output

Run on 31 October 2026, two Lisbon hotels, three Friday check-ins in December, two nights each:

```bash
python compare_dates.py "Lumen Hotel Lisbon" "Brown's Avenue Hotel Lisbon" \
    --checkin 2026-12-04 2026-12-11 2026-12-18 --nights 2 --currency EUR
```

```text
hotel                                2026-12-04   2026-12-11   2026-12-18
Lumen Hotel & The Lisbon Light Sh         91.81         77.5       74.13*
Brown's | Avenue Hotel                   138.07      133.16*      133.16*
6 rows -> hotel_prices_by_date.csv (prices per night, EUR)
```

Each of the three runs took about 5 seconds. For the first hotel, the 18 December weekend is about 19% cheaper per night than 4 December. The CSV shows where the price comes from (booking site lists shortened):

```csv
checkIn,hotel,pricePerNight,total,cheapestSite,officialSite,cheaperThanOfficial,sites,url
2026-12-04,Lumen Hotel & The Lisbon Light Show,91.81,183.62,Vio.com,110.01,Vio.com | goseek.com | Etrip.net | Stayforlong.com | ...,23,https://www.google.com/travel/hotels/entity/ChoIkamS5OLO9d2QARoNL2cvMTFrODM4N24zORAB
2026-12-04,Brown's | Avenue Hotel,138.07,276.14,goseek.com,142.9,goseek.com | Vio.com | Etrip.net | dealbase.com,18,https://www.google.com/travel/hotels/entity/ChkIhLCQwvjO2IYWGg0vZy8xMXE0bTZieDkyEAE
2026-12-11,Brown's | Avenue Hotel,133.16,266.32,Brown's Avenue,133.16,,20,https://www.google.com/travel/hotels/entity/ChkIhLCQwvjO2IYWGg0vZy8xMXE0bTZieDkyEAE
2026-12-18,Lumen Hotel & The Lisbon Light Show,74.13,148.26,dealbase.com,83.98,dealbase.com | Vio.com | Etrip.net | ...,20,https://www.google.com/travel/hotels/entity/ChoIkamS5OLO9d2QARoNL2cvMTFrODM4N24zORAB
```

Two things stand out. Google listed 18 to 23 booking sites per hotel and date. And the cheapest site is not always the same: on 4 December four sites undercut Brown's own website (142.90), while on 11 and 18 December the official site was the cheapest. That `cheaperThanOfficialSite` column is a ready-made rate-parity check if you run a hotel.

Each item also carries `matchedHotel` (the name you passed) next to `hotelName` (what Google matched), ratings, review counts, star class, amenities, coordinates and the full `bookingSources` list with a booking link per site.

## Variants

**One-click examples.** Prefilled public tasks you can run from the Apify console: [Paris hotel prices](https://apify.com/lagenuina_agency/google-hotels-scraper/examples/paris-hotel-prices), [New York hotel prices](https://apify.com/lagenuina_agency/google-hotels-scraper/examples/new-york-hotel-prices), [London hotel prices](https://apify.com/lagenuina_agency/google-hotels-scraper/examples/london-hotel-prices) and [Rome hotel prices](https://apify.com/lagenuina_agency/google-hotels-scraper/examples/rome-hotel-prices).

**Track prices over time.** Schedule one run a day with relative dates (`"checkInDate": "14 days"`) so every run looks the same distance ahead. Each row has `previousPricePerNight` and `priceChangePct`; set `onlyPriceChanges: true` with a `monitorStoreName` to receive only hotels whose price changed (unchanged hotels are not charged).

**Price alerts, JavaScript and n8n.** The example repo [assa83/google-hotels-prices-api](https://github.com/assa83/google-hotels-prices-api) has an alert script ("tell me when this hotel drops under 150 a night"), a Node.js version and an n8n workflow.

**AI agents (MCP).** Give the actor to Claude, Cursor or any MCP client:

```json
{
  "mcpServers": {
    "google-hotels-scraper": {
      "url": "https://mcp.apify.com/?actors=lagenuina_agency/google-hotels-scraper"
    }
  }
}
```

Then ask "find 4-star hotels in Lisbon under €150 for 12 to 14 December" and the agent maps it to location, dates, filters and currency.

## What it costs

Pay per hotel, with every booking site's price included: $3.99 per 1,000 hotels on the Apify Free plan, down to $2.39 on Gold and above, no start fee. Hotels removed by your filters are not charged. The six hotel-date rows above cost about two and a half cents on the Free plan. Check the [actor page](https://apify.com/lagenuina_agency/google-hotels-scraper) for current prices.

## Limits worth knowing

- **One room per search.** `adults` plus `childrenAges`; multi-room bookings are not supported.
- **One date pair per run.** That is why the script starts one run per date.
- **Prices follow the market you choose.** Booking sites and whether taxes are included follow Google's display for `countryCode` and `currency`, so set both explicitly and keep them the same across runs.
- **Hotels given by name are matched to Google's first result.** Check `matchedHotel` against `hotelName`, or use the Google Hotels URL or `entityId` from the CSV for an exact property.
- **No room types, cancellation policies or review texts.** You get nightly and total prices per site, ratings and review statistics.
- **Up to 500 hotels per location**, and never more than Google itself lists for that search.

## Wrap-up

Without a public Google Hotels API, a hosted scraper plus one Python file is the shortest path from "what does this hotel cost on these dates?" to a CSV you can sort. Copy the script, set `APIFY_TOKEN`, put in your hotels and dates, and schedule it if you want to watch prices move.

---

*Disclosure: I built this actor. It is published on Apify by Quelli di Carne Genuina, and I earn from its usage.*
