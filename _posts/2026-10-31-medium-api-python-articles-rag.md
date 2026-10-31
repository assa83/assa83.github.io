---
title: "Medium API in Python: get articles by tag or author, full text for RAG"
description: "Medium's official API never let you read articles by tag, author or search, and RSS stops at 10 stories. Here is a tested Python script that pulls Medium articles with claps and full Markdown text, and chunks them into JSONL for a RAG index."
actor: medium-scraper
tags: python, medium, ai, rag
---

Medium is one of the largest piles of practical, recent writing on programming, data and AI. That makes it an obvious source for a RAG knowledge base, a topic monitor ("what are people publishing about LLM evaluation this week?") or a content-research spreadsheet.

The usual ways in are disappointing:

1. **The official Medium API is not for reading.** It was built for publishing to your own account; it never offered "articles by tag", "articles by author" or search, and it no longer accepts new integrations.
2. **RSS is shallow.** `medium.com/feed/tag/rag` works, but it lists only the latest ~10 stories, without claps, responses or reading time, and with HTML you have to clean before an LLM can use it.
3. **Scraping the page means fighting the layout.** Medium's article HTML is full of navigation, sign-up prompts and generated class names.

This tutorial uses the [Medium Scraper actor on Apify](https://apify.com/lagenuina_agency/medium-scraper) as the API: articles by tag, author, publication or keyword, with engagement metrics and the full public text as clean Markdown. Then one Python file turns them into chunks ready for an embedding model.

## What you need

- Python 3.9+
- A free Apify account and your API token, from [Settings → Integrations](https://console.apify.com/settings/integrations)

```bash
export APIFY_TOKEN=your_token_here
pip install "apify-client>=1.7,<2"
```

No Medium account or token is involved. The examples target the 1.x client, where `.call()` returns a plain dict.

## The short version: one function

```python
import os
from apify_client import ApifyClient

client = ApifyClient(os.environ["APIFY_TOKEN"])


def medium_articles(tag, n=10):
    run = client.actor("lagenuina_agency/medium-scraper").call(
        run_input={"tags": [tag], "maxItemsPerFeed": n}
    )
    return client.dataset(run["defaultDatasetId"]).list_items().items


for a in medium_articles("rag", n=3):
    print(a["publishedAt"][:10], a["claps"], a["readingTimeMinutes"], a["isPaywalled"], a["title"])
```

Output, run on 31 October 2026:

```text
2026-10-31 0 5.7 False Apple Foundation Models Isn’t Stupid
2026-10-31 0 15.1 False Build an Agentic Knowledge Graph From Scratch: No Graph DB, No Extraction Model
2026-10-31 0 1.3 False AI Hallucination vs RAG: How Retrieval-Augmented Generation Reduces Incorrect AI Answers
```

Tags return the newest stories first, which is why everything here was published today and has zero claps so far. Each item also has `markdown` and `text` (the full body), `subtitle`, `author` and `authorUsername`, `publication`, `responses`, `wordCount`, `tags`, `imageUrl` and `url`.

Instead of `tags` you can pass `users` (`"@netflixtechblog"`), `publications` (`"towards-data-science"`), `searchQueries` (`"rag agents"`) or `articleUrls`, in any combination.

## The full script: Medium articles to chunked JSONL for RAG

A RAG index wants small pieces of text with enough metadata to cite the source. The script below fetches articles, skips member-only stories (only Medium's short public preview is available for those), splits each Markdown body into chunks of about 1,500 characters on paragraph boundaries, and appends them to a JSONL file. If the file already exists, articles that are in it are skipped, so you can re-run it on a schedule as a topic monitor.

```python
"""Medium articles -> chunked JSONL for a RAG index (no extra dependencies).

Usage: python to_rag.py --tag rag --max 20
       python to_rag.py --user @netflixtechblog --max 10
       python to_rag.py --search "llm evaluation" --max 15
Re-running appends only articles not already in the file.
"""
import argparse
import json
import os
import re
import sys

from apify_client import ApifyClient

ACTOR = "lagenuina_agency/medium-scraper"
TOKEN = os.environ.get("APIFY_TOKEN")
if not TOKEN:
    sys.exit("Set APIFY_TOKEN first: https://console.apify.com/settings/integrations")


def chunks(markdown, size=1500):
    """Split on paragraphs, then pack paragraphs into chunks of about `size` characters."""
    out, cur = [], ""
    for para in re.split(r"\n{2,}", markdown):
        if cur and len(cur) + len(para) > size:
            out.append(cur.strip())
            cur = ""
        cur += para + "\n\n"
    if cur.strip():
        out.append(cur.strip())
    return out


def main():
    p = argparse.ArgumentParser()
    p.add_argument("--tag", action="append", default=[])
    p.add_argument("--user", action="append", default=[])
    p.add_argument("--search", action="append", default=[])
    p.add_argument("--max", type=int, default=20, help="max articles per source")
    p.add_argument("--out", default="medium_chunks.jsonl")
    a = p.parse_args()
    if not (a.tag or a.user or a.search):
        p.error("give at least one --tag, --user or --search")

    seen = set()
    if os.path.exists(a.out):
        with open(a.out, encoding="utf-8") as f:
            seen = {json.loads(line)["article_id"] for line in f}

    client = ApifyClient(TOKEN)
    run = client.actor(ACTOR).call(run_input={
        "tags": a.tag, "users": a.user, "searchQueries": a.search, "maxItemsPerFeed": a.max,
    })
    added = skipped = n_chunks = 0
    with open(a.out, "a", encoding="utf-8") as f:
        for art in client.dataset(run["defaultDatasetId"]).iterate_items():
            if art["id"] in seen:
                continue
            if art["isPaywalled"]:  # only Medium's public preview: not worth indexing
                skipped += 1
                continue
            for i, text in enumerate(chunks(art["markdown"] or "")):
                f.write(json.dumps({
                    "id": f"{art['id']}-{i}", "article_id": art["id"], "chunk": i,
                    "title": art["title"], "author": art["author"], "url": art["url"],
                    "published": art["publishedAt"], "tags": art["tags"], "claps": art["claps"],
                    "text": text,
                }, ensure_ascii=False) + "\n")
                n_chunks += 1
            added += 1
            print(f"  + {art['readingTimeMinutes']:>4} min  {art['title'][:70]}")
    print(f"{added} articles, {n_chunks} chunks -> {a.out} ({skipped} member-only skipped)")


if __name__ == "__main__":
    main()
```

The input fields that matter:

| Field | Meaning |
|---|---|
| `tags` | Medium tags, newest stories first |
| `users` | Usernames with or without `@`, or profile URLs |
| `publications` | Publication slugs or URLs, custom domains included |
| `searchQueries` | Keyword search, like the search box on medium.com |
| `maxItemsPerFeed` | Max articles per tag, author, publication or search |
| `maxItems` | Hard cap across all sources |
| `includeContent` | Full text as `markdown` and `text` (on by default; off = metadata only) |

## Real output

Run on 31 October 2026, the four latest posts of the Netflix Technology Blog:

```bash
python to_rag.py --user @netflixtechblog --max 4
```

```text
  +  8.8 min  Beyond API Compatibility: Building a Tool to Measure the Blast Radius
  + 13.5 min  Netflix Stratum Media Processing: Automated Container Right-Sizing
  + 12.3 min  Reliable GPU compute for large training & inference jobs
  + 13.6 min  Making Failure Diagnosis Reliable with Pensive AI
4 articles, 71 chunks -> medium_chunks.jsonl (0 member-only skipped)
```

The run took about 5 seconds. The first line of `medium_chunks.jsonl` (text shortened):

```json
{"id": "9474f2fd4f90-0", "article_id": "9474f2fd4f90", "chunk": 0, "title": "Beyond API Compatibility: Building a Tool to Measure the Blast Radius of Library Changes", "author": "Netflix Technology Blog", "url": "https://netflixtechblog.com/beyond-api-compatibility-building-a-tool-to-measure-the-blast-radius-of-library-changes-9474f2fd4f90", "published": "2026-10-23T16:01:03.846Z", "tags": ["java", "software-engineering", "api-design", "developer-tools", "code-analysis"], "claps": 84, "text": "By [Roberto Perez Alcolea](https://www.linkedin.com/in/robertoperezalcolea/), Java Platform Team\n\n![Quasar connects a proposed API change with observed references in consumer repositories, ...](https://miro.medium.com/v2/resize:fit:1400/1*K06WZmnRhhXBboIskWb8qg.png)\n\nAt Netflix, shared Java libraries support applications developed across a large, independently evolving repository ecosystem. ..."}
```

The Markdown keeps links, image alt text and code blocks, so chunks still read well on their own and an answer can cite `url` and `title`. The blog lives on a custom domain (netflixtechblog.com) and the `@netflixtechblog` username still works.

## From JSONL to an index

Every vector store accepts this shape. With [Chroma](https://www.trychroma.com/), for example, it is a few lines (it embeds with its default model on first use; `pip install chromadb`):

```python
import json
import chromadb

rows = [json.loads(line) for line in open("medium_chunks.jsonl", encoding="utf-8")]
col = chromadb.PersistentClient("medium_db").get_or_create_collection("medium")
col.upsert(
    ids=[r["id"] for r in rows],
    documents=[r["text"] for r in rows],
    metadatas=[{"title": r["title"], "url": r["url"], "published": r["published"]} for r in rows],
)
print(col.query(query_texts=["how does Netflix right-size containers?"], n_results=2)["metadatas"])
```

On the 71 chunks above (the first run downloads Chroma's 79 MB default embedding model), both top hits come from the right article:

```text
[[{'url': 'https://netflixtechblog.com/netflix-stratum-media-processing-automated-container-right-sizing-ddf9aa68a801', 'title': 'Netflix Stratum Media Processing: Automated Container Right-Sizing', 'published': '2026-09-25T16:01:02.879Z'}, {'url': 'https://netflixtechblog.com/netflix-stratum-media-processing-automated-container-right-sizing-ddf9aa68a801', 'published': '2026-09-25T16:01:02.879Z', 'title': 'Netflix Stratum Media Processing: Automated Container Right-Sizing'}]]
```

Swap in pgvector, Qdrant or your provider's embeddings as you like; the chunk `id` makes upserts idempotent.

## Variants

**One-click examples.** Prefilled public tasks you can run from the Apify console: [Medium AI articles](https://apify.com/lagenuina_agency/medium-scraper/examples/medium-ai-articles), [one author's Medium articles](https://apify.com/lagenuina_agency/medium-scraper/examples/medium-author-articles), [Medium programming articles](https://apify.com/lagenuina_agency/medium-scraper/examples/medium-programming-articles) and [Medium data science articles](https://apify.com/lagenuina_agency/medium-scraper/examples/medium-data-science-articles).

**Monitor a topic.** Schedule `python to_rag.py --tag llm --max 30` daily. Thanks to the `seen` check, the file grows only by new stories, and the printed lines are your daily digest. Note that every scraped article is billed, including the ones the script skips, so keep `--max` close to the number of stories a tag gets per day.

**Markdown files instead of JSONL.** The example repo [assa83/medium-scraper-api](https://github.com/assa83/medium-scraper-api) writes one `.md` file per article with YAML front matter, which suits Obsidian or a docs-based RAG loader. It also has a Node.js version and an n8n workflow.

**AI agents and LangChain.** Through the Apify MCP server, an agent can call the actor directly:

```json
{
  "mcpServers": {
    "medium-scraper": {
      "url": "https://mcp.apify.com/?actors=lagenuina_agency/medium-scraper"
    }
  }
}
```

Then ask "summarise the 10 most recent Medium articles tagged rag". In LangChain, `langchain-apify`'s `ApifyDatasetLoader` can map `markdown` to `page_content` and `url` to metadata.

## What it costs

Pay per article, platform usage included, no start fee: $8 per 1,000 articles on the Apify Free plan, down to $5.60 on Gold and above. Failed or not-found articles are not billed. The seven articles in my two test runs come to under six cents on the Free plan. Set `includeContent: false` if you only need titles, claps and dates (same price, smaller dataset). Check the [actor page](https://apify.com/lagenuina_agency/medium-scraper) for current prices.

## Limits worth knowing

- **No paywall bypass.** Member-only stories come with `isPaywalled: true` and only the preview Medium shows logged-out readers. The script skips them; you may prefer to keep their metadata.
- **Fresh stories have no claps yet.** Tags return the newest stories first, so engagement numbers are near zero for the latest ones. For "most clapped" lists, scrape a wider window and sort, or use authors and publications.
- **Responses are counted, not scraped.** No comment text, highlights or follower counts.
- **Embeds become links.** YouTube videos, Gists and tweets inside an article are returned as links, not content.
- **`feedUrls` and tags without an archive fall back to RSS**, which lists only the ~10 latest stories.
- **The text belongs to its authors.** Indexing for your own retrieval is one thing; republishing is another. Check Medium's terms and the authors' rights for your use.

## Wrap-up

Medium's API will not hand you articles by tag or author, and RSS stops at ten. A hosted scraper gives you hundreds of stories per source with clean Markdown, and fifty lines of Python turn them into citation-ready chunks. Copy the script, set `APIFY_TOKEN` and point it at the tag, author or search your knowledge base needs.

---

*Disclosure: I built this actor. It is published on Apify by Quelli di Carne Genuina, and I earn from its usage.*
