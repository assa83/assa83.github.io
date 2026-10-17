---
title: "Google Flights API in Python: track fares and get price alerts"
description: "Google has no public Flights API since QPX Express shut down. Here is a tested Python script that checks a route every day, logs the cheapest fare to CSV and alerts you when it drops below your target."
actor: google-flights-scraper
tags: python, travel, api, automation
---

There is no official Google Flights API. Google shut down QPX Express, the last public flight search API it offered, in 2018. What is left:

- **Airline and GDS APIs** (Amadeus, Duffel, Sabre and the like) are built for booking. They need an account, sometimes a contract, and their prices are not always the ones you see on Google Flights.
- **Price-alert apps**, including Google's own "Track prices" toggle, send you an email. They do not give you the numbers to put in a spreadsheet, a dashboard or a script.
- **Scraping Google Flights yourself** means reverse-engineering a JavaScript-heavy page that changes often.

If what you want is "check LHR → JFK every morning, keep a history and ping me when it is cheap", a hosted scraper and a short Python script are enough. This tutorial uses the [Google Flights Scraper actor on Apify](https://apify.com/lagenuina_agency/google-flights-scraper) and the official `apify-client` package.

By the end you will have `fare_tracker.py`, which on each run:

1. finds the cheapest fare on your route N days ahead,
2. finds the cheapest departure day in the following week,
3. appends one row to `fares_history.csv`,
4. prints an `ALERT` line when the fare is at or below your target.

## What you need

- Python 3.9+
- A free Apify account and your API token, from [Settings → Integrations](https://console.apify.com/settings/integrations)

```bash
export APIFY_TOKEN=your_token_here
pip install "apify-client>=1.7,<2"
```

## The script

I wrote this for the post and ran it today against the live actor. The example repo [assa83/google-flights-api](https://github.com/assa83/google-flights-api) has a sibling script that prints every fare instead of logging the cheapest.

```python
"""Track Google Flights fares for one route and alert when the price drops.

Usage: python fare_tracker.py LHR JFK --days-ahead 30 --max-price 450
Run it once a day (cron, GitHub Actions, n8n): every run appends to fares_history.csv.
"""
import argparse
import csv
import datetime
import os
import sys

from apify_client import ApifyClient

ACTOR = "lagenuina_agency/google-flights-scraper"
TOKEN = os.environ.get("APIFY_TOKEN")
if not TOKEN:
    sys.exit("Set APIFY_TOKEN first: https://console.apify.com/settings/integrations")
HISTORY = "fares_history.csv"


def main():
    p = argparse.ArgumentParser()
    p.add_argument("origin", help="IATA airport code, e.g. LHR")
    p.add_argument("destination", help="IATA airport code, e.g. JFK")
    p.add_argument("--days-ahead", type=int, default=30)
    p.add_argument("--max-price", type=float, help="alert when the cheapest fare is at or below this")
    p.add_argument("--calendar-days", type=int, default=7, help="also find the cheapest day in the next N days")
    p.add_argument("--currency", default="USD")
    a = p.parse_args()

    client = ApifyClient(TOKEN)
    run = client.actor(ACTOR).call(run_input={
        "origin": a.origin, "destination": a.destination,
        "departureDate": f"{a.days_ahead} days",  # relative: a daily run always looks N days ahead
        "currency": a.currency, "sortBy": "price", "maxResults": 5,
        "calendarDays": a.calendar_days,
    })
    items = list(client.dataset(run["defaultDatasetId"]).iterate_items())
    for e in (i for i in items if i["type"] == "error"):
        sys.exit(f"actor error: {e['error']}")

    flights = [i for i in items if i["type"] == "flight" and i.get("price") is not None]
    if not flights:
        sys.exit("no fares found")
    best = min(flights, key=lambda f: f["price"])
    level = (best.get("priceInsights") or {}).get("priceLevel", "")
    days = [i for i in items if i["type"] == "calendarDay" and i.get("lowestPrice") is not None]
    cheapest_day = min(days, key=lambda d: d["lowestPrice"]) if days else None

    new_file = not os.path.exists(HISTORY)
    with open(HISTORY, "a", newline="", encoding="utf-8") as f:
        w = csv.writer(f)
        if new_file:
            w.writerow(["checked_at", "route", "departure_date", "price", "currency", "airlines", "stops", "price_level"])
        w.writerow([datetime.datetime.now().isoformat(timespec="minutes"), f"{a.origin}-{a.destination}",
                    best["departureDate"], best["price"], best["currency"], " / ".join(best["airlines"]),
                    best["stops"], level])

    print(f"Cheapest {a.origin}->{a.destination} on {best['departureDate']}: {best['price']} {best['currency']} "
          f"({', '.join(best['airlines'])}, {best['stops']} stop(s)), Google says '{level}'")
    if cheapest_day:
        print(f"Cheapest day in the next {a.calendar_days}: {cheapest_day['departureDate']} "
              f"at {cheapest_day['lowestPrice']} {cheapest_day['currency']}")
    if a.max_price is not None and best["price"] <= a.max_price:
        print(f"ALERT: {best['price']} {best['currency']} is at or below your {a.max_price:g} target")


if __name__ == "__main__":
    main()
```

The input fields used:

| Field | Meaning |
|---|---|
| `origin`, `destination` | IATA airport codes |
| `departureDate` | `YYYY-MM-DD` or a relative value like `"30 days"`, so a scheduled run keeps looking the same distance ahead |
| `sortBy: "price"`, `maxResults` | Cheapest itineraries first, keep 5 |
| `calendarDays` | Also return the lowest fare for each of the next N departure days |
| `currency` | Any ISO code; add `market` (e.g. `GB`) to see fares as offered in another country |

The output mixes row types: `flight` rows (itineraries), `calendarDay` rows (one per date) and `error` rows. The script splits them by `type`.

## Real output

Run on 17 October 2026, London Heathrow to New York JFK, 30 days ahead, with a $550 target:

```bash
python fare_tracker.py LHR JFK --days-ahead 30 --max-price 550 --calendar-days 7
```

```text
INFO  LHR→JFK 2026-11-16: 5 itineraries
INFO  Done: 12 row(s), 0 route error(s).
Status: SUCCEEDED
Cheapest LHR->JFK on 2026-11-16: 505 USD (JetBlue, 0 stop(s)), Google says 'typical'
Cheapest day in the next 7: 2026-11-17 at 404 USD
ALERT: 505 USD is at or below your 550 target
```

About ten seconds per run. After two runs, `fares_history.csv` looks like this:

```csv
checked_at,route,departure_date,price,currency,airlines,stops,price_level
2026-10-17T15:35,LHR-JFK,2026-11-16,505,USD,JetBlue,0,typical
2026-10-17T15:39,LHR-JFK,2026-11-16,505,USD,JetBlue,0,typical
```

Two things in that output are worth more than the price itself:

- **Flying one day later saves $101.** The calendar found $404 on 17 November against $505 on the 16th. Most alert apps only watch the date you typed.
- **Google's own verdict comes with each fare.** Every `flight` row has a `priceInsights` block. Shortened:

```json
{
  "price": 505,
  "airlines": ["JetBlue"],
  "flightNumbers": ["B6 2220"],
  "departureTime": "2026-11-16T08:05",
  "durationMinutes": 489,
  "stops": 0,
  "co2EmissionsKg": 549,
  "priceInsights": {
    "lowestPrice": 505,
    "typicalPriceLow": 400,
    "typicalPriceHigh": 570,
    "priceLevel": "typical",
    "priceHistory": [{ "date": "2026-08-16", "price": 365 }, "..."]
  },
  "bookingUrl": "https://www.google.com/travel/flights/booking?tfs=..."
}
```

`priceHistory` holds about 60 days of Google's tracked prices for the search. Here it shows the same flight at $365 in mid-August, so "typical" today is still well above where it started. A sensible alert rule uses this: fire when the price is `low`, or below `typicalPriceLow`, instead of a number you guessed.

## Running it every day

The script appends, so scheduling it is all you need to build a history:

- **cron**: `0 7 * * * cd /path/to/tracker && APIFY_TOKEN=... python fare_tracker.py LHR JFK --max-price 450`
- **GitHub Actions**: a `schedule:` workflow that runs the script and commits `fares_history.csv` back to the repo. Store the token as a repository secret.
- **Turn the ALERT line into a message**: pipe the output through `grep ALERT` and into your email or Slack webhook of choice.

## Variants

**n8n, no code.** The example repo has an [n8n workflow](https://github.com/assa83/google-flights-api/tree/main/n8n): every day at 07:00 it searches the route 30 days ahead, a Code node keeps only fares at or below `MAX_PRICE`, and you attach a Slack or Gmail node. Import the JSON, add a Header Auth credential (`Authorization: Bearer <token>`) and set your route.

**Many routes at once.** The actor takes a `routes` list (`[{"origin": "LHR", "destination": "JFK", "departureDate": "2026-12-10"}, ...]`), so one run can watch every route you care about.

**AI agents (MCP).** Give the actor to Claude, Cursor or any MCP client:

```json
{
  "mcpServers": {
    "google-flights-scraper": {
      "url": "https://mcp.apify.com/?actors=lagenuina_agency/google-flights-scraper"
    }
  }
}
```

Then ask "what is the cheapest day to fly Milan to New York in the first week of December?" and the agent runs the calendar search.

## What it costs

The actor charges per row: one itinerary or one calendar day. The run above returned 12 rows (5 flights and 7 calendar days) and cost a fraction of a cent. Until 26 October 2026 the price is $0.16 per 1,000 rows on the Apify Free plan ($0.08 on Gold); from 27 October it is $0.80 on Free and $0.56 on Gold. At the new price a daily check like this one is about 4,400 rows a year, roughly $3.50 on the Free plan. Check the [actor page](https://apify.com/lagenuina_agency/google-flights-scraper) for current prices.

## Limits worth knowing

- **What Google shows, not every fare.** Each search returns the itineraries on Google's results page, typically 8 to 60, not the full "View more flights" list.
- **No multi-city trips**, no "Explore anywhere" and no per-seller prices (Expedia, Kiwi and so on).
- **Round trips** return outbound itineraries with the round-trip total. Turn on `resolveReturnFlights` to get the matching return flights.
- **Prices depend on `market` and `currency`.** The same flight can cost differently when searched from the US or the UK. Keep both fixed in a tracker, or the history is not comparable.
- **The calendar costs one row per day.** A 60-day sweep is 60 rows; use `calendarOnly` if you do not need itineraries.
- The data is what Google Flights displays publicly. The scraper is not affiliated with Google, and you should book on the airline's or agent's site, not on a scraped price.

## Wrap-up

Without a public Google Flights API, a small daily script on top of a hosted scraper gets you the part that matters: a fare history you own and an alert that fires on your rules. Copy `fare_tracker.py`, set `APIFY_TOKEN`, pick a route and schedule it.

---

*Disclosure: I built this actor. It is published on Apify by Quelli di Carne Genuina, and I earn from its usage.*
