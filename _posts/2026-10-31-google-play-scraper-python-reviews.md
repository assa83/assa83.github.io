---
title: "Google Play scraper in Python: app details and reviews to CSV"
description: "Google's Play Developer API only covers your own apps. Here is a tested Python script that gets any app's ratings, installs and user reviews from Google Play into a CSV, with a quick look at what 1-star reviewers complain about."
actor: google-play-scraper
tags: python, googleplay, android, webscraping
---

Product managers, ASO consultants and indie developers keep asking the same two questions about apps they do not own: how is this app doing (rating, installs, last update), and what are its users complaining about?

Google Play shows both on every app page. Getting them into code is less obvious:

1. **The official API is for your own apps.** The Google Play Developer API returns reviews of apps in your developer account, not a competitor's.
2. **Open-source libraries work until they don't.** The `google-play-scraper` packages are fine for a few calls from your laptop; at scale you add proxies, retries and parallelism yourself.
3. **Reviews are paginated and sorted.** You need hundreds of them, in one language and country, ideally filtered by star rating, and in a format a spreadsheet or an LLM can read.

This tutorial uses the [Google Play Store Scraper actor on Apify](https://apify.com/lagenuina_agency/google-play-scraper) and keeps the Python short: one function for app details, then a script that exports reviews to CSV and shows what unhappy users write about.

## What you need

- Python 3.9+
- A free Apify account and your API token, from [Settings → Integrations](https://console.apify.com/settings/integrations)

```bash
export APIFY_TOKEN=your_token_here
pip install "apify-client>=1.7,<2"
```

No Google account is involved. The examples target the 1.x client, where `.call()` returns a plain dict.

## The short version: app details in one function

```python
import os
from apify_client import ApifyClient

client = ApifyClient(os.environ["APIFY_TOKEN"])


def play_app(app_id, country="us", language="en"):
    run = client.actor("lagenuina_agency/google-play-scraper").call(run_input={
        "appIdsOrUrls": [app_id],
        "countryCode": country,
        "language": language,
    })
    return client.dataset(run["defaultDatasetId"]).list_items().items[0]


app = play_app("com.duolingo")
for k in ["title", "developer", "score", "ratings", "installs", "version", "updatedAt", "genre", "priceText"]:
    print(f"{k:<10} {app[k]}")
```

Output, run on 31 October 2026:

```text
title      Duolingo: Learn Languages
developer  Duolingo
score      4.746547
ratings    51463857
installs   500,000,000+
version    VARY
updatedAt  2026-10-26T19:44:49.000Z
genre      Education
priceText  Free
```

`appIdsOrUrls` takes package names (`com.duolingo`) or full Play Store URLs. `VARY` is what Google Play reports when the version "varies with device". The same item also carries the ratings histogram, `minInstalls` as a number, the description, `recentChanges`, the developer's public email, website and address, and screenshot URLs.

## The full script: reviews to CSV, with a quick sentiment check

Turn on `includeReviews` and the same run returns one extra item per review (`type: "review"`) next to the app item (`type: "app"`).

```python
"""Google Play reviews to CSV, with a quick star breakdown and the words people complain about.

Usage: python reviews.py com.duolingo --reviews 500
       python reviews.py com.spotify.music --reviews 300 --stars 1 --country gb
"""
import argparse
import csv
import os
import re
import sys
from collections import Counter

from apify_client import ApifyClient

ACTOR = "lagenuina_agency/google-play-scraper"
TOKEN = os.environ.get("APIFY_TOKEN")
if not TOKEN:
    sys.exit("Set APIFY_TOKEN first: https://console.apify.com/settings/integrations")

STOP = set("""a an and are as at be but by can could did do does dont don't for from get got had has have i i'm
if in is it it's its just me my no not now of on or so that the this to too was we were what when with
would you your app very all one even after they them there their than then only also been more will
""".split())


def main():
    p = argparse.ArgumentParser()
    p.add_argument("app", help="package name (com.duolingo) or Play Store URL")
    p.add_argument("--reviews", type=int, default=200)
    p.add_argument("--sort", default="newest", choices=["newest", "relevant", "rating"])
    p.add_argument("--stars", type=int, choices=[1, 2, 3, 4, 5], help="only reviews with this rating")
    p.add_argument("--country", default="us")
    p.add_argument("--language", default="en")
    a = p.parse_args()

    run_input = {"appIdsOrUrls": [a.app], "countryCode": a.country, "language": a.language,
                 "includeReviews": True, "maxReviewsPerApp": a.reviews, "reviewSort": a.sort}
    if a.stars:
        run_input["reviewScore"] = a.stars
    client = ApifyClient(TOKEN)
    run = client.actor(ACTOR).call(run_input=run_input)
    items = client.dataset(run["defaultDatasetId"]).list_items().items
    app = next(i for i in items if i["type"] == "app")
    reviews = [i for i in items if i["type"] == "review"]

    out = f"reviews_{app['appId']}.csv"
    with open(out, "w", newline="", encoding="utf-8") as f:
        cols = ["date", "score", "text", "thumbsUp", "version", "replyText", "reviewId"]
        w = csv.DictWriter(f, fieldnames=cols, extrasaction="ignore")
        w.writeheader()
        w.writerows(reviews)

    print(f"{app['title']}: {app['score']:.2f} stars from {app['ratings']:,} ratings, {app['installs']} installs")
    stars = Counter(r["score"] for r in reviews)
    for s in range(5, 0, -1):
        print(f"  {s}* {stars[s]:>4}  {'#' * round(40 * stars[s] / max(len(reviews), 1))}")
    replied = sum(1 for r in reviews if r.get("replyText"))
    print(f"  developer replied to {replied} of {len(reviews)}")

    low = [r["text"] or "" for r in reviews if r["score"] <= 2]
    words = Counter(w for t in low for w in set(re.findall(r"[a-z']{3,}", t.lower())) if w not in STOP)
    print(f"most common words in {len(low)} low (1-2*) reviews:", ", ".join(f"{w} ({n})" for w, n in words.most_common(8)))
    print(f"{len(reviews)} reviews -> {out}")


if __name__ == "__main__":
    main()
```

The review fields in the input:

| Field | Meaning |
|---|---|
| `includeReviews` | Also return reviews, one dataset item each |
| `maxReviewsPerApp` | How many reviews per app |
| `reviewSort` | `newest`, `relevant` or `rating` |
| `reviewScore` | Keep only one star rating, e.g. `1` for complaints |
| `countryCode`, `language` | The Play Store market and language the reviews come from |

The CSV deliberately leaves out `userName` and `userImage`: you rarely need reviewer identities to analyse feedback, and not storing them keeps you clear of personal-data questions.

## Real output

Run on 31 October 2026, the 40 newest US English reviews of Duolingo:

```bash
python reviews.py com.duolingo --reviews 40
```

```text
Duolingo: Learn Languages: 4.75 stars from 51,463,823 ratings, 500,000,000+ installs
  5*   29  #############################
  4*    8  ########
  3*    2  ##
  2*    0
  1*    1  #
  developer replied to 0 of 40
most common words in 1 low (1-2*) reviews: thank (1), free (1), man (1), dualingo (1), few (1), permission (1), money (1), less (1)
40 reviews -> reviews_com.duolingo.csv
```

The run took about 13 seconds. The newest reviews mirror the overall rating: almost all 5 stars, mostly short ("its a good app", "very nice app 👍"). With one negative review in forty, the word count is meaningless. To learn what is going wrong you have to ask for the complaints directly:

```bash
python reviews.py com.duolingo --reviews 40 --stars 1
```

```text
Duolingo: Learn Languages: 4.75 stars from 51,463,822 ratings, 500,000,000+ installs
  5*    0
  4*    0
  3*    0
  2*    0
  1*   40  ########################################
  developer replied to 0 of 40
most common words in 40 low (1-2*) reviews: language (7), much (6), lesson (6), ads (5), time (5), bad (5), like (5), free (4)
40 reviews -> reviews_com.duolingo.csv
```

This one took about a minute, because Google serves reviews newest-first and the actor reads pages until it has 40 one-star reviews. A few rows of the CSV:

```csv
date,score,text,thumbsUp,version,replyText,reviewId
2026-10-30T04:11:28.165Z,1,"too much ads. if I want to do a 4 min lesson, I have to watch 3 unskippable video ads.",0,6.103.6,,28e690b7-949c-4f8e-a218-7c4f97cf248f
2026-10-30T03:54:49.439Z,1,so many bugs,0,6.103.6,,3c1c04a9-7218-450c-a8cd-6f53e898c032
```

"ads", "lesson" and "time" together point at one theme you can verify by reading those rows: ads interrupting short lessons. A word count is a crude tool, but it tells you which reviews to read first. The `version` column lets you check whether complaints cluster on one release.

## Going further: let an LLM read them

Forty reviews you can read yourself; four thousand you cannot. The CSV is already the right shape for an LLM: send the `text` column in batches with a prompt such as "group these reviews into at most 8 complaint themes, with a count and two quotes each". Keep `reviewId` so every quote can be traced back to its row.

## Variants

**One-click examples.** Prefilled public tasks you can run from the Apify console: [ChatGPT app reviews](https://apify.com/lagenuina_agency/google-play-scraper/examples/chatgpt-app-reviews), [TikTok app reviews](https://apify.com/lagenuina_agency/google-play-scraper/examples/tiktok-app-reviews), [Instagram app reviews](https://apify.com/lagenuina_agency/google-play-scraper/examples/instagram-app-reviews) and [top free games on Google Play](https://apify.com/lagenuina_agency/google-play-scraper/examples/top-free-games-google-play).

**Search, charts and competitors.** The same actor takes `searchQueries` (apps ranking for a keyword), `developerIds` (every app of a developer), `similarToAppIds` and top charts (`includeCollection` with `collectionChart` and `collectionCategory`). Mix them with `includeReviews` to get reviews of every app found.

**JavaScript and n8n.** The example repo [assa83/google-play-scraper-api](https://github.com/assa83/google-play-scraper-api) has a keyword-to-CSV script in Python and Node.js, and an n8n workflow.

**AI agents (MCP).** Give the actor to Claude, Cursor or any MCP client:

```json
{
  "mcpServers": {
    "google-play-scraper": {
      "url": "https://mcp.apify.com/?actors=lagenuina_agency/google-play-scraper"
    }
  }
}
```

Then ask "summarise the last 300 one-star reviews of com.spotify.music" and the agent runs the scrape and the summary in one go.

## What it costs

Pay per result, platform usage included, no start fee: $1.84 per 1,000 apps and $0.92 per 1,000 reviews on the Apify Free plan, down to $0.96 and $0.48 on Gold and $0.45 and $0.22 on Diamond. The three test runs above (three app lookups and 80 reviews) come to about eight cents on the Free plan. Check the [actor page](https://apify.com/lagenuina_agency/google-play-scraper) for current prices.

## Limits worth knowing

- **The language filter is not strict.** With `language: "en"`, at least one review in another language (Portuguese) still came back in my 1-star run. Filter again on your side if it matters.
- **Star filters take longer.** Google has no server-side star filter, so `reviewScore` reads pages until enough matching reviews are found: about 1 minute instead of 13 seconds in the runs above, more for rare ratings.
- **About 30 apps per search keyword.** That is all Google Play shows for one search; use several keywords, developers or charts for more.
- **Not returned:** app size, the data safety section, permissions. No login, so no private or paid-only data.
- **Apps unavailable in the chosen country** fail for that country (they are listed in a `FAILED` record and not billed).
- **Reviews are user content.** Use them for analysis; be careful with reviewer names and quoting individuals publicly.

## Wrap-up

Google Play's own API stops at your own apps, but the data you want about competitors is public. One function gives you an app's vital signs; one script gives you its reviews in a CSV and a first idea of what users are unhappy about. Copy the script, set `APIFY_TOKEN` and point it at the app you care about.

---

*Disclosure: I built this actor. It is published on Apify by Quelli di Carne Genuina, and I earn from its usage.*
