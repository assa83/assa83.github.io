---
title: "Weekly Google Trends report in Google Sheets with n8n (no code)"
description: "Import one n8n workflow and get Google Trends interest for your keywords appended to a Google Sheet every Monday morning."
actor: google-trends-scraper
tags: n8n, googletrends, automation, nocode
---

Checking Google Trends by hand works for a one-off question. It falls apart when you want to follow the same ten keywords every week: you forget, the date range shifts, and the screenshots don't line up.

This post sets up a small n8n workflow that does it for you. Every Monday at 08:00 it:

1. Fetches Google Trends interest over time for your keywords
2. Flattens the result into one row per keyword and date
3. Appends those rows to a Google Sheet

You import a ready-made workflow, so you don't write code. The data comes from the [Google Trends Scraper actor on Apify](https://apify.com/lagenuina_agency/google-trends-scraper), which n8n calls with a plain HTTP Request node.

**Be aware before you start:** the workflow file is valid n8n JSON and its Apify call uses the same endpoint and input that I tested from Python. I have **not** run this exact workflow inside an n8n instance. Node versions differ between n8n releases, so treat it as a starting point and do one manual run before you activate it.

## What you need

- An n8n instance (n8n Cloud or self-hosted)
- A Google account with access to Google Sheets
- A free Apify account and your API token from [Settings → Integrations](https://console.apify.com/settings/integrations)
- The workflow file: [`n8n/workflow.json`](https://github.com/assa83/google-trends-api-python/blob/main/n8n/workflow.json) from the repo [assa83/google-trends-api-python](https://github.com/assa83/google-trends-api-python)

## How the workflow is built

Four nodes in a straight line:

```text
Schedule Trigger ──> Run Apify actor ──> Flatten timeline ──> Append to Google Sheets
(Mon 08:00)          (HTTP Request)      (Code)               (Google Sheets)
```

- **Schedule Trigger**: weekly, on day 1 (Monday), at hour 8.
- **Run Apify actor**: a `POST` to Apify's `run-sync-get-dataset-items` endpoint. This endpoint starts the actor, waits for it to finish and returns the dataset items in the response, so there is no polling. The node timeout is set to 300,000 ms (5 minutes), the most this endpoint waits.
- **Flatten timeline**: a short JavaScript snippet that turns each keyword's nested `timeline` array into separate rows.
- **Append to Google Sheets**: adds those rows to the sheet, mapping fields to columns automatically.

## Step 1: Import the workflow

1. Download `workflow.json` from the repo.
2. In n8n, open **Workflows**, then use **Import from File** and pick the file. You can also open an empty workflow and paste the JSON onto the canvas.
3. You should see the four nodes connected. Two of them will show credential warnings. That is expected, you set them up next.

## Step 2: Create the Apify credential

The HTTP Request node expects a **Header Auth** credential named `Apify API token`.

1. Open the **Run Apify actor** node.
2. Under *Authentication* it is already set to *Generic Credential Type* → *Header Auth*. In the credential dropdown, choose **Create new credential**.
3. Fill it in:
   - **Credential name**: `Apify API token`
   - **Name**: `Authorization`
   - **Value**: `Bearer ` followed by your token (with the space after `Bearer`)
4. Save and make sure the node now uses this credential.

The token lives in n8n's encrypted credential store, not in the workflow JSON. If you export or share the workflow later, the token does not go with it.

## Step 3: Set your keywords

Still in **Run Apify actor**, edit the JSON body:

```json
{
  "searchTerms": ["chatgpt", "gemini"],
  "geo": "US",
  "timeRange": "today 3-m",
  "includeInterestByRegion": false
}
```

- `searchTerms`: your keywords. Each one comes back as one result.
- `geo`: a country or region code (`US`, `GB`, `IT`, …). Use `""` for worldwide.
- `timeRange`: the same syntax as the Trends website, such as `today 3-m` or `today 12-m`.

The other input fields, including related queries and Trending Now, are on the [actor page](https://apify.com/lagenuina_agency/google-trends-scraper) (Input tab).

Keep the keyword list modest: the synchronous endpoint waits up to 300 seconds. If a long list times out, switch to Apify's asynchronous `/runs` endpoint with a webhook.

## Step 4: Connect Google Sheets

1. Create a new Google Sheet. In row 1 of the first tab, write these headers exactly:

   | A | B | C | D |
   |---|---|---|---|
   | searchTerm | geo | date | value |

   The node uses automatic mapping, which matches incoming field names to column headers. Without the headers it has nothing to map to.

2. Copy the sheet ID from its URL. It is the long string between `/d/` and `/edit`.
3. Open **Append to Google Sheets**:
   - Create or select a **Google Sheets** credential (n8n walks you through the Google sign-in).
   - Replace `YOUR_GOOGLE_SHEET_ID` with your sheet ID.
   - Check that *Sheet* matches your tab name. The workflow uses `Sheet1`, so change it if your tab has another name, for example in a localised Google account.

## Step 5: Test once, by hand

Before scheduling anything, click **Execute workflow** (or **Test workflow**, depending on your n8n version) and check each node:

- **Run Apify actor** should output one item per keyword, each with `searchTerm`, `geo` and a `timeline` array.
- **Flatten timeline** should output many small items such as:

  ```json
  { "searchTerm": "chatgpt", "geo": "US", "date": "2026-10-16", "value": 84 }
  ```

- **Append to Google Sheets** should report the rows it wrote, and they should be in your sheet.

If the HTTP node returns `401`, the header value is wrong. Check that it reads `Bearer <token>` with one space and no quotes. If Sheets complains about columns, check the header row from Step 4.

## Step 6: Activate the schedule

Flip the workflow to **Active**. The trigger fires every Monday at 08:00 in your n8n instance's timezone. Check that under the workflow or instance settings if your server runs in UTC and you don't.

To change the day or hour, open **Schedule Trigger** and edit the weekly rule.

## One thing to decide: overlapping weeks

Each Monday the workflow fetches the **last three months**, not just the last week. Appending blindly means most rows show up again every week. You have three reasonable options:

1. **Keep it as a snapshot log.** Add a column with the run date and treat each week's block as a separate snapshot. This is useful because Google Trends rescales values as new data arrives, so a snapshot shows what the numbers looked like at that moment.
2. **Keep only recent days.** Change the Code node to keep only the last 7 days, so each run adds just the new week. For example:

   ```js
   // One row per keyword and day, only the last 7 days.
   const since = new Date(Date.now() - 7 * 864e5).toISOString().slice(0, 10);
   const rows = [];
   for (const { json } of $input.all()) {
     for (const p of json.timeline ?? []) {
       const date = p.date.slice(0, 10);
       if (date >= since) rows.push({ json: { searchTerm: json.searchTerm, geo: json.geo, date, value: p.value } });
     }
   }
   return rows;
   ```

   This snippet is a variant of the one in the repo and I have not run it in n8n, so check its output in a manual test first.
3. **Deduplicate in Sheets.** Leave the workflow as is and use *Data → Data cleanup → Remove duplicates* now and then, or build your report on a `UNIQUE()` range.

Also remember that the latest point is usually **partial** (still being collected). The original API response marks it with `isPartial: true`. The Code node in the workflow does not copy that field, so add `isPartial: p.isPartial` to the row object if you want to filter it out in Sheets.

## Reading the numbers

Google Trends values are **relative**, from 0 to 100, where 100 is the peak for that query, region and time range. They are not search volumes. In my tests each keyword came back on its own 0–100 scale, so compare how each line moves over time rather than the absolute values between two keywords.

For a quick report, build a pivot table (`date` as rows, `searchTerm` as columns) and add a line chart on it.

## What it costs

n8n costs whatever your plan or server costs. On the Apify side the actor uses pay-per-result pricing with no start fee. One keyword with all its data is one result. The price per 1,000 results goes from **$2.40 on the Free plan down to $0.24 on Gold**. Ten keywords every Monday is 520 results a year. Check the [actor page](https://apify.com/lagenuina_agency/google-trends-scraper) for current prices.

## Prefer code?

The same repo has a Python script that writes the data to dated CSV files (`trends_YYYY-MM-DD.csv` and `rising_YYYY-MM-DD.csv`) and a Node.js example. I covered the Python version in a separate post. If you work with AI agents, you can also expose the actor as an MCP tool at `https://mcp.apify.com/?actors=lagenuina_agency/google-trends-scraper`.

The scraper reads only publicly available data and is not affiliated with or endorsed by Google. Use it in line with the applicable terms and laws.

---

*Disclosure: I built this actor. It is published on Apify by Quelli di Carne Genuina, and I earn from its usage.*
