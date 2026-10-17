---
title: "YouTube transcript API in Python that works on servers (no IP blocks)"
description: "youtube-transcript-api works on your laptop and gets blocked on AWS, GCP or Render. Here is a tested Python script that fetches YouTube transcripts from any server, as text and SRT."
actor: youtube-transcript-scraper
tags: python, youtube, api, ai
---

If you have ever built something on top of YouTube transcripts (a summariser, a RAG index, a search over a podcast archive), you probably started with the open-source [`youtube-transcript-api`](https://github.com/jdepoix/youtube-transcript-api) package. It is a good library, and on your laptop it works on the first try.

Then you deploy it. On AWS, Google Cloud, Azure, Render, Railway or a GitHub Actions runner, the same call fails with `RequestBlocked` or `IpBlocked`. YouTube blocks most cloud and datacenter IP ranges for caption requests, and the library's own README says so: on a server you need rotating residential proxies.

So the real problem is not parsing captions. It is getting a request through from a server IP, every day, without running your own proxy pool.

This tutorial moves that part to a hosted scraper, the [YouTube Transcript Scraper actor on Apify](https://apify.com/lagenuina_agency/youtube-transcript-scraper), and keeps your Python small. The code below runs anywhere `pip install` works, because the request to YouTube does not leave from your server.

## What you need

- Python 3.9+
- A free Apify account and your API token, from [Settings → Integrations](https://console.apify.com/settings/integrations)

```bash
export APIFY_TOKEN=your_token_here
pip install "apify-client>=1.7,<2"
```

The examples target the 1.x client, where `.call()` returns a plain dict.

## The short version: one function

If you only need the text of one video inside your app, this is all it takes:

```python
import os
from apify_client import ApifyClient

client = ApifyClient(os.environ["APIFY_TOKEN"])


def get_transcript(url, languages=("en",)):
    run = client.actor("lagenuina_agency/youtube-transcript-scraper").call(
        run_input={"youtube_url": url, "languages": list(languages)}
    )
    item = client.dataset(run["defaultDatasetId"]).list_items().items[0]
    if not item.get("success"):
        raise RuntimeError(item.get("error_message", "no transcript"))
    return item["non_timestamped"]


print(get_transcript("https://youtu.be/jNQXAC9IVRw")[:80])
```

Output:

```text
All right, so here we are, in front of the elephants the cool thing about these
```

`languages` is an ordered preference list: the first language YouTube has wins, and manual captions beat auto-generated ones in the same language.

## The full script: many videos to .txt, .srt and a CSV

For batches (a list of URLs, a playlist or a whole channel), this is the script from the example repo [assa83/youtube-transcript-scraper-api](https://github.com/assa83/youtube-transcript-scraper-api). I ran it today, unchanged.

```python
"""Download YouTube transcripts (videos, playlists or a channel) to text files + one CSV.

Usage:
  python example.py https://www.youtube.com/watch?v=dQw4w9WgXcQ
  python example.py https://www.youtube.com/playlist?list=PL... --max 20
  python example.py --channel @TED --max 10 --lang en es
"""
import argparse
import csv
import os
import re
import sys

from apify_client import ApifyClient

ACTOR = "lagenuina_agency/youtube-transcript-scraper"
TOKEN = os.environ.get("APIFY_TOKEN")
if not TOKEN:
    sys.exit("Set APIFY_TOKEN first: https://console.apify.com/settings/integrations")


def main():
    p = argparse.ArgumentParser()
    p.add_argument("urls", nargs="*", help="video / Shorts / playlist URLs or video IDs")
    p.add_argument("--channel", help="@handle or channel URL: transcribe its newest videos")
    p.add_argument("--max", type=int, default=10, help="max videos per channel or playlist")
    p.add_argument("--lang", nargs="+", default=["en"], help="preferred languages, in order")
    p.add_argument("--out", default="transcripts", help="output folder")
    a = p.parse_args()
    if not a.urls and not a.channel:
        p.error("pass at least one URL or --channel")

    run_input = {"youtube_url": a.urls, "languages": a.lang, "max_videos": a.max, "output_formats": ["srt"]}
    if a.channel:
        run_input.update(channel=a.channel, channel_transcripts=True)
    run = ApifyClient(TOKEN).actor(ACTOR).call(run_input=run_input)
    items = ApifyClient(TOKEN).dataset(run["defaultDatasetId"]).list_items().items

    os.makedirs(a.out, exist_ok=True)
    with open(os.path.join(a.out, "transcripts.csv"), "w", newline="", encoding="utf-8") as f:
        w = csv.writer(f)
        w.writerow(["video_id", "title", "channel", "language", "source_type", "words", "url", "error"])
        for it in items:
            w.writerow([it.get("video_id"), it.get("title"), it.get("channel_name"), it.get("language_code"),
                        it.get("source_type"), it.get("word_count"), it.get("url"), it.get("error_message", "")])
            if it.get("success"):
                name = re.sub(r"[^\w-]+", "_", f"{it['video_id']}_{it.get('title') or ''}")[:80]
                with open(os.path.join(a.out, name + ".txt"), "w", encoding="utf-8") as t:
                    t.write(f"{it.get('title')}\n{it['url']}\n\n{it['non_timestamped']}\n")
                with open(os.path.join(a.out, name + ".srt"), "w", encoding="utf-8") as t:
                    t.write(it["srt"])
    ok = sum(1 for it in items if it.get("success"))
    print(f"{ok} transcript(s), {len(items) - ok} without captions/failed -> {a.out}/")


if __name__ == "__main__":
    main()
```

The input fields that matter:

| Field | Meaning |
|---|---|
| `youtube_url` | One URL or a list: videos, Shorts, `youtu.be` links, playlists or bare video IDs |
| `channel` + `channel_transcripts` | A `@handle` or channel URL; with `channel_transcripts: true` its newest videos are transcribed |
| `max_videos` | How many videos per channel or playlist |
| `languages` | Ordered language preference |
| `output_formats` | Extra formats: `srt`, `vtt`, `text` |

## Real output

Run on 17 October 2026:

```bash
python example.py https://www.youtube.com/watch?v=jNQXAC9IVRw https://www.youtube.com/watch?v=dQw4w9WgXcQ
```

The run log shows exactly the problem this post is about, handled for you:

```text
INFO  2 video(s) to process (languages: en)
INFO  YouTube is blocking this IP range, switching to residential proxy.
INFO  [Status message]: 2 transcript(s)
Status: SUCCEEDED, Message: 2 transcript(s)
2 transcript(s), 0 without captions/failed -> transcripts/
```

The whole run took about 12 seconds. `transcripts/transcripts.csv`:

```csv
video_id,title,channel,language,source_type,words,url,error
dQw4w9WgXcQ,Rick Astley - Never Gonna Give You Up (Official Video) (4K Remaster),Rick Astley,en,Manual,487,https://www.youtube.com/watch?v=dQw4w9WgXcQ,
jNQXAC9IVRw,Me at the zoo,jawed,en,Manual,39,https://www.youtube.com/watch?v=jNQXAC9IVRw,
```

And the start of the `.srt` file for the second video:

```text
1
00:00:01,360 --> 00:00:03,040
[♪♪♪]

2
00:00:18,640 --> 00:00:21,880
♪ We're no strangers to love ♪
```

Each dataset item also carries `timestamped` segments (`text`, `start`, `duration`) if you want to cite the exact second a phrase was said, which is handy for RAG answers that link back to the video.

## Variants

**A whole channel.** `python example.py --channel @TED --max 10 --lang en es` transcribes the ten newest videos of a channel. Listing the channel is free; you pay only for transcripts.

**n8n, no code.** The example repo has an [n8n workflow](https://github.com/assa83/youtube-transcript-scraper-api/tree/main/n8n) that runs every morning, transcribes the newest 5 videos of a channel and appends title, URL, language, word count and transcript to Google Sheets. Import the JSON, add a Header Auth credential with `Authorization: Bearer <token>`, set your channel and sheet.

**AI agents (MCP).** Give the actor to Claude, Cursor or any MCP client as a tool:

```json
{
  "mcpServers": {
    "youtube-transcript-scraper": {
      "url": "https://mcp.apify.com/?actors=lagenuina_agency/youtube-transcript-scraper"
    }
  }
}
```

Then ask "summarise the last three videos of @channel" and the agent fetches the transcripts itself.

## What it costs

Pay per transcript, proxies included, no start fee: $2.20 per 1,000 transcripts on the Apify Free plan, $1.60 on Gold. A three-hour podcast costs the same as a Short. Videos without captions and failed videos are free. The two-video run above cost less than a cent. Check the [actor page](https://apify.com/lagenuina_agency/youtube-transcript-scraper) for current prices.

## Limits worth knowing

- **Captions only.** The actor reads the captions YouTube already has (manual or auto-generated). A video with no captions returns an error row, not a transcript. There is no speech-to-text.
- **Translation is best effort.** YouTube refuses most automated translation requests; when it does, you get the original language plus a `translation_error` field.
- **No signed-in content.** Private, members-only and age-restricted videos fail, because the actor never uses an account.
- **Live streams in progress** and premieres that have not aired have no transcript yet.
- **Latency.** A hosted run adds a few seconds of start-up compared with a direct library call. For a nightly batch that does not matter; for a user waiting on a button, fetch ahead of time or cache.

## Wrap-up

If `youtube-transcript-api` works locally and breaks in production, the fix is not more retry code: it is sending the request from an IP YouTube accepts. Clone the [example repo](https://github.com/assa83/youtube-transcript-scraper-api), set `APIFY_TOKEN` and run it from the same server that was getting blocked.

---

*Disclosure: I built this actor. It is published on Apify by Quelli di Carne Genuina, and I earn from its usage.*
