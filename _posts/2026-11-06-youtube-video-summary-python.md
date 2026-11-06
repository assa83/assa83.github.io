---
title: "YouTube video summary in Python: a timestamped outline, no LLM needed"
description: "Summarise any YouTube video into a clickable outline of key moments with a tested Python script. It runs on any server, needs no LLM and costs a fifth of a cent per video."
actor: youtube-transcript-scraper
tags: python, youtube, ai, nlp
---

People search for "YouTube summary" more than for "YouTube transcript": over the last 12 months Google Trends puts the two at 29 and 25 on the same chart, and "summary of youtube video" and "youtube summary ai" are among its top related searches. The usual answer is a browser extension or a website. That is fine for one video. It does not help if you want summaries inside your own tool, for a list of videos, or running on a server every morning.

Doing it in code has three catches:

1. **The transcript is the hard part.** The open-source `youtube-transcript-api` works on your laptop and gets blocked from cloud servers (I covered that in [YouTube transcript API in Python that works on servers](/youtube-transcript-api-python/)).
2. **LLM summaries cost money and drop the timestamps.** A one-hour talk is about 12,000 words. Sending it to a model is cheap once and adds up across hundreds of videos, and what comes back usually no longer says where in the video each point was made.
3. **A paragraph is often the wrong format.** Before watching a one-hour talk you want to know what it covers and where to jump, so a list of moments you can click beats a neat paragraph.

This tutorial builds that list. The [YouTube Transcript Scraper actor on Apify](https://apify.com/lagenuina_agency/youtube-transcript-scraper) fetches the captions, and about 40 lines of standard-library Python pick the passage that best represents each part of the video. No model, no extra API key, same result every run.

## What you need

- Python 3.9+
- A free Apify account and your API token, from [Settings → Integrations](https://console.apify.com/settings/integrations)

```bash
export APIFY_TOKEN=your_token_here
pip install "apify-client>=1.7,<2"
```

The examples target the 1.x client, where `.call()` returns a plain dict.

## What the actor returns

Each video comes back as one item. `non_timestamped` is the plain text, and `timestamped` is the part we need here, a list of caption lines with their start time in seconds:

```python
[{"text": "hi everyone so recently I gave a", "start": 0.16, "duration": 4.08},
 {"text": "30-minute talk on large language models", "start": 2.28, "duration": 4.119}]
```

The same item has `title`, `channel_name`, `total_seconds`, `duration_human`, `word_count` and `source_type` (`Manual` or `Auto-generated`).

## The script

```python
"""Summarise a YouTube video into a timestamped outline, without an LLM.

Usage: python summarize.py https://youtu.be/UF8uR6Z6KLc
       python summarize.py https://youtu.be/zjkBMFhNj_g --sections 8 --lang en
"""
import argparse
import math
import os
import re
import sys
from collections import Counter

from apify_client import ApifyClient

ACTOR = "lagenuina_agency/youtube-transcript-scraper"
TOKEN = os.environ.get("APIFY_TOKEN")
if not TOKEN:
    sys.exit("Set APIFY_TOKEN first: https://console.apify.com/settings/integrations")

STOP = set("""a about actually after all also am an and any are around as at back be because been before being
but by can could did do does doing don't down even every first for from get getting go going gonna good got had
has have he her here him his how i i'm i've if in into is it it's its just kind know last let like little look
lot make many maybe me might more most much my need new no not now of off ok okay on one only or other our out
over people really right said same say see she should so some something still such take than that that's the
their them then there these they they're thing things think this those through time to too two uh um up us use
very want was way we we're well were what when where which while who why will with would yeah you you're your
""".split())


def words(text):
    return [w for w in re.findall(r"[a-z][a-z'-]+", text.lower()) if w not in STOP]


def chunks(snippets, size=40):
    """Glue caption lines into passages of about `size` words, ending on a full stop when there is one."""
    out, text, start = [], [], None
    for s in snippets:
        start = s["start"] if start is None else start
        text.append(s["text"].replace("\n", " "))
        n = len(" ".join(text).split())
        if n >= size * 1.5 or (n >= size * 0.6 and text[-1].rstrip().endswith((".", "?", "!"))):
            out.append({"start": start, "text": " ".join(text)})
            text, start = [], None
    if text:
        out.append({"start": start, "text": " ".join(text)})
    return out


def outline(item, sections):
    passages = chunks(item["timestamped"])
    df = Counter(w for p in passages for w in set(words(p["text"])))
    idf = {w: math.log(len(passages) / n) for w, n in df.items()}
    length = item["total_seconds"] / sections
    for i in range(sections):
        part = [p for p in passages if i * length <= p["start"] < (i + 1) * length]
        if not part:
            continue
        # A word matters if it is frequent in this section and rare in the rest of the video.
        weight = Counter(w for p in part for w in words(p["text"]))
        weight = {w: n * idf[w] for w, n in weight.items()}
        top = sorted(weight, key=weight.get, reverse=True)[:4]

        def score(p):
            ws = set(words(p["text"]))
            return sum(weight[w] for w in ws) / (len(p["text"].split()) + 5)

        yield max(part, key=score), top


def main():
    p = argparse.ArgumentParser()
    p.add_argument("url")
    p.add_argument("--sections", type=int, help="default: one per 2 minutes, between 4 and 12")
    p.add_argument("--lang", nargs="+", default=["en"])
    a = p.parse_args()

    client = ApifyClient(TOKEN)
    run = client.actor(ACTOR).call(run_input={"youtube_url": a.url, "languages": a.lang})
    item = client.dataset(run["defaultDatasetId"]).list_items().items[0]
    if not item.get("success"):
        sys.exit(item.get("error_message", "no transcript"))

    sections = a.sections or min(12, max(4, round(item["total_seconds"] / 120)))
    print(f"# {item['title']}\n{item['channel_name']}, {item['duration_human']}, {item['word_count']:,} words, "
          f"{item['source_type'].lower()} captions\n")
    for best, top in outline(item, sections):
        s = int(best["start"])
        print(f"- [{s // 60}:{s % 60:02d}](https://youtu.be/{item['video_id']}?t={s}) "
              f"**{', '.join(top)}**: {best['text']}")


if __name__ == "__main__":
    main()
```

How it works:

- **`chunks()`** glues caption lines into passages of about 40 words. Human-made captions have punctuation, so passages end on a full stop. Auto-generated ones have none, so they are cut by length.
- **`outline()`** splits the video into equal sections of time. In each section it weights every word by how often it appears there and how rare it is in the rest of the video: that is TF-IDF, with one section playing the part of the document. The four heaviest words become the label.
- **The passage** with the most weight per word is the one quoted. Dividing by length stops long passages from winning just for being long.

## Real output

Run on 6 November 2026 on a 15-minute speech with human-made captions, 18 seconds from start to finish:

```bash
python summarize.py https://youtu.be/UF8uR6Z6KLc
```

```text
# Steve Jobs' 2005 Stanford Commencement Address
Stanford, 15m 4s, 2,298 words, manual captions

- [0:35](https://youtu.be/UF8uR6Z6KLc?t=35) **mother, college, adopted, stories**: Truth be told I never graduated from college and this is the closest I've ever gotten to a college graduation. Today I want to tell you three stories from my life. That's it.
- [3:04](https://youtu.be/UF8uR6Z6KLc?t=184) **college, idea, didn't, classes**: It wasn't all romantic. I didn't have a dorm room, so I slept on the floor in friends' rooms, I returned coke bottles for the 5 cent deposits to buy food with, and I would walk the 7 miles across town every Sunday night to get one good meal a week at the Hare Krishna temple. I loved it.
- [5:02](https://youtu.be/UF8uR6Z6KLc?t=302) **connect, never, dots, looking**: But it was very, very clear looking backwards ten years later. Again, you can't connect the dots looking forward; you can only connect them looking backwards.
- [7:31](https://youtu.be/UF8uR6Z6KLc?t=451) **company, fired, apple, began**: During the next five years, I started a company named NeXT, another company named Pixar, and fell in love with an amazing woman who would become my wife.
- [7:39](https://youtu.be/UF8uR6Z6KLc?t=459) **work, better, great, keep**: Pixar went on to create the worlds first computer animated feature film, Toy Story, and is now the most successful animation studio in the world.
- [9:40](https://youtu.be/UF8uR6Z6KLc?t=580) **means, cancer, pancreas, everything**: Remembering that I'll be dead soon is the most important tool I've ever encountered to help me make the big choices in life. Because almost everything all external expectations, all pride, all fear of embarrassment or failure - these things just fall away in the face of death, leaving only what is truly important.
- [12:42](https://youtu.be/UF8uR6Z6KLc?t=762) **death, old, living, closest**: Don't let the noise of others' opinions drown out your own inner voice. And most important, have the courage to follow your heart and intuition.
- [14:23](https://youtu.be/UF8uR6Z6KLc?t=863) **stay, hungry, foolish, google**: And now, as you graduate to begin anew, I wish that for you. Stay Hungry. Stay Foolish. Thank you all very much. The preceding program is copyrighted by Stanford University.
```

Read the labels on their own and you have the speech: adopted, college, connecting the dots, fired from Apple, cancer, death, stay hungry. Every timestamp opens the video at that second.

The same script on a one-hour talk with auto-generated captions, asking for 6 sections (20 seconds, first three shown):

```bash
python summarize.py https://youtu.be/zjkBMFhNj_g --sections 6
```

```text
# [1hr Talk] Intro to Large Language Models
Andrej Karpathy, 59m 50s, 12,151 words, auto-generated captions

- [1:37](https://youtu.be/zjkBMFhNj_g?t=97) **parameters, network, neural, file**: file and the Run uh some kind of a code that runs those parameters so the parameters are basically the weights or the parameters of this neural network that is the language model we'll go into that in a bit because this is a 70 billion parameter model uh every one of those parameters is stored as 2 bytes and so
- [16:21](https://youtu.be/zjkBMFhNj_g?t=981) **stage, documents, assistant, training**: conversations and they should be very high quality conversations and fundamentally people create them based on abling instructions so we swap out the data set now and we train on these Q&A documents we uh and this process is called fine tuning once you do this you obtain what we call an assistant model so this assistant model now subscribes
```

No punctuation, "um" and "uh" left in, and a word like "parameters" repeated: this is what auto-captions look like. The labels hold up well and the quotes are readable, but they are raw speech. Further down the same output the labels read `operating, large, language, there's` at 44:20 and `attack, google, prompt, doc` at 53:10, which are the talk's "LLM as an operating system" and security parts.

Two things in these outputs are worth seeing as they are, because they are the method's weak spots:

- The sections are cut by the clock and not by topic. At 7:39 in the speech the label says `work, better, great, keep` and the quote is about Pixar: the section starts in the Apple story and ends on "keep looking", and the best passage happens to be in the first half. A section can also pick a passage right next to the one before (7:31 and 7:39).
- The labels are words, not phrases. `there's` slipped through the stop list in one run; add such words to `STOP` when you see them.

## Going further: hand it to an LLM

This outline is extractive: every sentence is a real quote, so it cannot invent anything, but it cannot rephrase either. If you want a written summary, `non_timestamped` is ready to send to any model, and the outline makes a good cheap first pass: send only the videos worth reading in full. I am not printing LLM code here because I only publish what I ran for the post, and I did not run that part.

The quickest way to get there without writing any of it is the MCP route below.

## Variants

**Other languages.** `--lang it en` takes Italian captions if the video has them and falls back to English. For another language, write its stop words into `STOP`, since the list above is English only. The input also has `translate_to` if you want YouTube to translate the captions first.

**A whole channel.** Pass `"channel": "@handle", "channel_transcripts": true, "max_videos": 10` instead of `youtube_url` and loop over the items. The script in the [transcript tutorial](/youtube-transcript-api-python/) shows that batch version.

**One-click examples.** Prefilled public tasks you can run from the Apify console: [one video](https://apify.com/lagenuina_agency/youtube-transcript-scraper/examples/youtube-video-transcript), [a channel](https://apify.com/lagenuina_agency/youtube-transcript-scraper/examples/youtube-channel-transcripts), [Lex Fridman's latest](https://apify.com/lagenuina_agency/youtube-transcript-scraper/examples/lex-fridman-transcripts) and [MKBHD's latest](https://apify.com/lagenuina_agency/youtube-transcript-scraper/examples/mkbhd-transcripts).

**JavaScript and n8n.** The example repo [assa83/youtube-transcript-scraper-api](https://github.com/assa83/youtube-transcript-scraper-api) has the transcript download in Python and Node.js, and an n8n workflow.

**AI agents (MCP).** Give the actor to Claude, Cursor or any MCP client:

```json
{
  "mcpServers": {
    "youtube-transcript": {
      "url": "https://mcp.apify.com/?actors=lagenuina_agency/youtube-transcript-scraper"
    }
  }
}
```

Then ask "summarise this video in five bullet points with timestamps" and the agent fetches the transcript itself.

## What it costs

Pay per transcript, platform usage and proxies included, no start fee: $0.0022 per video on the Apify Free plan, down to $0.0016 on Gold, whatever the length, so the one-hour talk costs the same as a Short. Videos without captions are free. The summary itself runs on your machine and costs nothing. Apify's free plan gives you $5 of credits a month, which is about 2,200 videos. Check the [actor page](https://apify.com/lagenuina_agency/youtube-transcript-scraper) for current prices.

## Limits worth knowing

- **No captions, no outline.** The actor reads captions and does not transcribe audio. Most videos have auto-generated ones, but not all.
- **Quotes, not prose.** The quality follows the captions: clean on human-made tracks, raw on auto-generated ones.
- **Tested on talks.** One person speaking on a clear subject suits this method. I have not measured it on interviews, vlogs or music, and it depends on words alone, so anything shown on screen is missed.
- **Short videos.** The default never goes below 4 sections, which can be too many for a short clip; pass `--sections 2`.
- **Stop words are English.** See the variants above.

## Wrap-up

A summary does not have to start with a model. Captions come with timestamps, and a bit of word counting turns them into a clickable outline that runs anywhere and costs nothing past the transcript. Copy the script, set `APIFY_TOKEN` and try it on the next long video you were not sure about watching.

---

*Disclosure: I built this actor. It is published on Apify by Quelli di Carne Genuina, and I earn from its usage.*
