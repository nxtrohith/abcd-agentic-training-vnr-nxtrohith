# Review Agent Automation (WhatsApp + Google Sheets)

Automate end‑to‑end product review analysis and AI summarization from an e‑commerce URL. This n8n workflow ingests a product URL, scrapes reviews and product details, combines signals, prompts an LLM to generate a buying‑guide style summary in Markdown, and delivers output to Google Sheets and WhatsApp.

- Two execution paths:
  - WhatsApp-triggered for ad‑hoc analysis.
  - Google Sheets-triggered for batch processing.
- Sources: Amazon product pages and reviews (via Apify Actor).
- Outputs: WhatsApp summary + Google Sheet rows (append or update).
- Models: Azure OpenAI or Google Gemini via LangChain Model Selector.

> Workflow JSON: [Automation UseCase/Review Agent Workflow (2).json](https://github.com/nxtrohith/abcd-agentic-training-vnr-nxtrohith/blob/fb78cb4a1f9902f1f6e4e31f2b478b0ace3fb8f9/Automation%20UseCase/Review%20Agent%20Workflow%20(2).json)

---

## Visuals

![Workflow Snapshot](https://github.com/nxtrohith/abcd-agentic-training-vnr-nxtrohith/blob/fb78cb4a1f9902f1f6e4e31f2b478b0ace3fb8f9/Automation%20UseCase/Workflow%20Snapshot.png)

WhatsApp Output:
![WhatsApp Output](https://github.com/nxtrohith/abcd-agentic-training-vnr-nxtrohith/blob/fb78cb4a1f9902f1f6e4e31f2b478b0ace3fb8f9/Automation%20UseCase/Workflow%20WhatsApp%20output.png)

Google Sheets Output:
![GSheet Output](https://github.com/nxtrohith/abcd-agentic-training-vnr-nxtrohith/blob/fb78cb4a1f9902f1f6e4e31f2b478b0ace3fb8f9/Automation%20UseCase/Workflow%20Gsheet%20output.png)

Demo videos:
- WhatsApp run: [Worflow WhatsApp Execution.mp4](https://github.com/nxtrohith/abcd-agentic-training-vnr-nxtrohith/blob/fb78cb4a1f9902f1f6e4e31f2b478b0ace3fb8f9/Automation%20UseCase/Worflow%20WhatsApp%20Execution.mp4)
- Google Sheets run: [Workflow GSheets Execution.mp4](https://github.com/nxtrohith/abcd-agentic-training-vnr-nxtrohith/blob/fb78cb4a1f9902f1f6e4e31f2b478b0ace3fb8f9/Automation%20UseCase/Workflow%20GSheets%20Execution.mp4)

---

## High‑Level Flow

Fetch Product Data → Collect Reviews → Process & Combine → Prompt AI → Append/Update Google Sheet → Send WhatsApp Summary

Two branches:

- WhatsApp URL → Apify + HTTP Scrape → Extract Details → Combine Reviews
  ↓
  AI Prompt (Reviewer)
  ↓
  Google Sheets Append
  ↓
  WhatsApp Reply (Generated Summary)

- Google Sheet (New URL) → Trigger → Scrape Product + Reviews
  ↓
  AI Prompt (AI Agent)
  ↓
  Google Sheet Update (Output column)

---

## Architecture

- n8n workflow with:
  - WhatsApp Trigger + Send (Cloud API)
  - Google Sheets Trigger + Tool (append/update)
  - Apify Actor (Amazon Reviews Scraper)
  - HTTP Request (product page HTML)
  - Code (JavaScript) nodes for parsing and aggregation
  - LangChain Agent + Model Selector (Azure OpenAI / Gemini)

ASCII overview:

```
[WhatsApp Trigger] ──► [Apify Reviews] ─┐
                          └──────────────┼──► [Merge] ─► [Set] ─► [Combine Reviews] ─► [LangChain Reviewer] ──► [GSheets Append] ─► [WhatsApp Send]
[HTTP Product HTML] ──► [Parse Characteristics] ─┘

[GSheets Trigger] ─► [Filter] ─► [Get Rows] ─► [HTTP HTML] ─► [Parse Characteristics] ─┐
                                                                  [Apify Reviews] ─────┼──► [Merge1] ─► [Split Out] ─► [Set1] ─► [Combine Reviews1] ─► [LangChain AI Agent] ─► [GSheets Update] ─► [Loop]
```

---

## Nodes and Responsibilities

### Branch 1 — WhatsApp‑Triggered Review Summary

1) WhatsApp Trigger
- Listens for incoming messages containing a product URL (`$json.messages[0].text.body`).
- Fan‑out to Review Scraper and Product Page Scraper.

2) Review Scraper (Apify Actor: Amazon Reviews Scraper)
- Parameters: `maxReviews=50`, `filterByRatings=all`, `productUrls=[WhatsApp URL]`.
- On failure: forwards to “Error Message” WhatsApp node with an alert.

3) Product Page Scraper (HTTP Request)
- Fetches raw HTML with browser‑like headers.

4) Characteristics Fetcher (Code)
- Parses HTML to extract title, description, specifications.
- Normalizes to:
  ```json
  {
    "title": "...",
    "description": "...",
    "specifications": ["Spec1: Value", "Spec2: Value"],
    "input": "Product URL"
  }
  ```

5) Merge
- Combines Apify reviews with parsed product details using `input` (the URL) as the join key.

6) Sets Info (Set)
- Normalizes fields: `reviewDescription`, `stars`, `title`, `Product Description`, `URL`.

7) Combines Reviews (Code)
- Aggregates and computes:
  ```json
  {
    "product": "Product Title",
    "totalReviews": 47,
    "avgStars": 4.3,
    "url": "Product URL",
    "ProductDescription": "...",
    "combinedReviews": "Review1 --- Review2 --- Review3..."
  }
  ```

8) Reviewer (LangChain Agent)
- System prompt enforces: neutral, strictly review‑based, no hallucinations.
- Output sections:
  - Overview
  - Best for
  - Pros
  - Cons
  - Worth noting
  - Verdict

9) Google Sheets Tool (Append)
- Appends: Product Title, URL, Product Description, Average Rating, Review Summary.

10) WhatsApp Send
- Sends the generated Markdown summary back to the originating user.

---

### Branch 2 — Google Sheet‑Triggered Review Generator

1) Google Sheets Trigger
- Watches “Review Request Sheet” for changes in “Product Url”.

2) Filter
- Continues only if “Product Title” cell is empty (`=`).

3) Get Row(s) in Sheet
- Reads from “Review Request Sheet 2” where “Output” is blank.

4) Product Page Scraper1 (HTTP)
- Fetches raw HTML.

5) Review Scraper1 (Apify)
- Same parameters as Branch 1.

6) Characteristics Fetcher1 (Code)
- Extracts title, description, specifications.

7) Merge1
- Combines product details + reviews.

8) Split Out
- Splits reviews for batching.

9) Sets Info1 (Set)
- Normalizes fields.

10) Combines Reviews1
- Aggregates + averages.

11) AI Agent (LangChain + Model Selector)
- Inputs: product title, description, combined reviews, specifications.
- Instructs AI to generate a customer‑facing README‑style guide and “paste the response into the Output column of the Google Sheet.”

12) Google Sheets Tool (Update)
- Writes to “Output”, and updates Product Title and Review Description for the same row.

13) Loop Over Items
- Iterates over multiple rows.

14) Error Handling
- On scraping errors, sends WhatsApp alert: `Error running node 'Review Scraper'`.

---

## AI Model Selection

- Model Selector routes between:
  - Azure OpenAI (e.g., o4‑mini)
  - Google Gemini
- Context formatting and prompt assembly via LangChain nodes.
- Output constraint: structured Markdown matching the buying‑guide format.

---

## Example AI Output

```markdown
# Apple AirPods Pro (2nd Gen) — Should you buy it?

Overview: The AirPods Pro deliver excellent noise cancellation and comfort...

Who it’s best for:
- Frequent travelers
- iPhone users who value quick pairing
- ...

Pros:
- Great ANC performance
- Compact design
- ...

Cons:
- Battery life could be better
- Pricey compared to alternatives

Worth noting:
- Latest firmware improves connectivity
- Some users mention fit issues

Verdict: A premium option for those in the Apple ecosystem.
```

---

## Setup

### Prerequisites
- n8n (self‑hosted or cloud) with access to:
  - WhatsApp Cloud API credentials
  - Google Sheets API OAuth
  - Apify API token and access to actor “Amazon Reviews Scraper”
  - Azure OpenAI or Google Gemini API keys (LangChain integrations)
- A Google Sheet with:
  - “Review Request Sheet” and/or “Review Request Sheet 2”
  - Columns (suggested):
    - Product Url
    - Product Title
    - Product Description
    - Avg Rating
    - Review Description
    - Output

### Import the Workflow
- Download: [Review Agent Workflow (2).json](https://github.com/nxtrohith/abcd-agentic-training-vnr-nxtrohith/blob/fb78cb4a1f9902f1f6e4e31f2b478b0ace3fb8f9/Automation%20UseCase/Review%20Agent%20Workflow%20(2).json)
- In n8n, Import from file.
- Open the workflow and configure credentials on:
  - WhatsApp Trigger / WhatsApp Send
  - Google Sheets Trigger / Google Sheets Tool
  - Apify
  - Azure OpenAI and/or Google Gemini

### Configure IDs and Targets
- WhatsApp:
  - Phone Number ID, recipient strategy (dynamic or test number).
- Google Sheets:
  - Document URL/ID
  - Sheet name or GID for:
    - Append target (for WhatsApp branch)
    - Update target (for Sheets branch)
- Apify:
  - Actor: junglee/amazon-reviews-scraper (`R8WeJwLuzLZ6g4Bkk`)
  - Input configuration: `maxReviews`, `filterByRatings`, `productUrls`
- AI:
  - Set the default model in Model Selector or per‑Agent node.

---

## Key Code Patterns

HTML extraction (Characteristics Fetcher) – minimal example:
```js
// Input: item.json.body contains HTML string
const html = item.json.body || '';
const pick = (re) => (html.match(re)?.[1] || '').trim();

const title =
  pick(/id="productTitle"[^>]*>([^<]+)/i) ||
  pick(/<title>([^<]+)<\/title>/i);

const description =
  pick(/id="productDescription"[^>]*>([\s\S]*?)<\/div>/i) ||
  pick(/id="feature-bullets"[^>]*>([\s\S]*?)<\/div>/i);

const specs = [...html.matchAll(/<tr[^>]*>\s*<th[^>]*>(.*?)<\/th>\s*<td[^>]*>(.*?)<\/td>\s*<\/tr>/gis)]
  .map((m) => `${m[1].replace(/<[^>]+>/g,'').trim()}: ${m[2].replace(/<[^>]+>/g,'').trim()}`)
  .slice(0, 25);

return [
  {
    json: {
      title,
      description: description.replace(/<[^>]+>/g, ' ').replace(/\s+/g, ' ').trim(),
      specifications: specs,
      input: item.json.input || item.json.url,
    }
  }
];
```

Review aggregation (Combines Reviews):
```js
const items = $input.all();
const product = items.find(i => i.json.title)?.json.title || 'Unknown Product';

const reviews = items
  .map(i => (i.json.reviewDescription || '').trim())
  .filter(Boolean);

const stars = items
  .map(i => Number(i.json.stars))
  .filter((n) => Number.isFinite(n));

const avgStars = stars.length ? Number((stars.reduce((a,b) => a + b, 0) / stars.length).toFixed(2)) : null;

return [
  {
    json: {
      product,
      ProductDescription: items.find(i => i.json.description)?.json.description || '',
      url: items.find(i => i.json.URL)?.json.URL || items[0]?.json.input,
      totalReviews: reviews.length,
      avgStars,
      combinedReviews: reviews.join('\n---\n'),
    }
  }
];
```

Prompting considerations:
- System prompts instruct the model to:
  - Use only scraped content (no outside knowledge/hallucinations).
  - Emit a consistent README‑style Markdown structure.
- For Sheets branch: include instruction to write back to the “Output” column.

---

## Running

- WhatsApp path:
  1) Send a product URL to your connected WhatsApp number.
  2) n8n triggers, scrapes, summarizes, appends to GSheet, and replies with the Markdown summary.

- Google Sheets path:
  1) Add a row with a Product Url (leave Product Title and Output blank).
  2) Trigger detects change, scrapes, summarizes, and updates the same row’s Output (and Title/Description as configured).

---

## Data Model (Sheets)

- Append target (WhatsApp branch):
  - Product Title
  - Product Url
  - Product Description
  - Avg Rating
  - Review Description (AI summarized)

- Update target (Sheets branch):
  - Matching column: Product Url
  - Updated columns:
    - Product Title
    - Review Description
    - Output (full Markdown summary)

---

## Error Handling

- Apify/Review scraper failure:
  - WhatsApp alert: `Error running node 'Review Scraper'`.
- HTTP fetch errors:
  - Ensure request headers mimic a modern browser.
  - Consider retries/backoff or Apify proxy pools.
- Parsing gaps:
  - HTML structures vary; ensure selectors/regex are conservative.
  - Fallback to `<title>` or alternative blocks if `productTitle` missing.

---

## Performance and Reliability

- Rate limits:
  - Respect Apify and e‑commerce site limitations.
  - Use debounce/queueing for Sheets triggers to avoid flapping.
- Batching:
  - “Split Out” and “Loop Over Items” control throughput for bulk runs.
- Observability:
  - Enable n8n execution logging.
  - Add optional notification nodes (Slack/Email) for failures.

---

## Security and Compliance

- Do not commit secrets (API keys, tokens).
- Use n8n credentials store and environment variables.
- Review site terms of service and robots.txt before scraping.
- Avoid PII collection and comply with applicable data protection laws.

---

## Extensibility

- Add other marketplaces by swapping the Apify actor and parse logic.
- Enhance deduplication and URL normalization pre‑scrape.
- Store raw datasets to a data warehouse (e.g., BigQuery, Postgres) for auditability.
- Add vector embedding for semantic retrieval over review corpora.

---

## References

- Workflow JSON: [Automation UseCase/Review Agent Workflow (2).json](https://github.com/nxtrohith/abcd-agentic-training-vnr-nxtrohith/blob/fb78cb4a1f9902f1f6e4e31f2b478b0ace3fb8f9/Automation%20UseCase/Review%20Agent%20Workflow%20(2).json)
- WhatsApp demo: [Worflow WhatsApp Execution.mp4](https://github.com/nxtrohith/abcd-agentic-training-vnr-nxtrohith/blob/fb78cb4a1f9902f1f6e4e31f2b478b0ace3fb8f9/Automation%20UseCase/Worflow%20WhatsApp%20Execution.mp4)
- GSheets demo: [Workflow GSheets Execution.mp4](https://github.com/nxtrohith/abcd-agentic-training-vnr-nxtrohith/blob/fb78cb4a1f9902f1f6e4e31f2b478b0ace3fb8f9/Automation%20UseCase/Workflow%20GSheets%20Execution.mp4)
- Snapshots:
  - [Workflow Snapshot.png](https://github.com/nxtrohith/abcd-agentic-training-vnr-nxtrohith/blob/fb78cb4a1f9902f1f6e4e31f2b478b0ace3fb8f9/Automation%20UseCase/Workflow%20Snapshot.png)
  - [Workflow WhatsApp output.png](https://github.com/nxtrohith/abcd-agentic-training-vnr-nxtrohith/blob/fb78cb4a1f9902f1f6e4e31f2b478b0ace3fb8f9/Automation%20UseCase/Workflow%20WhatsApp%20output.png)
  - [Workflow Gsheet output.png](https://github.com/nxtrohith/abcd-agentic-training-vnr-nxtrohith/blob/fb78cb4a1f9902f1f6e4e31f2b478b0ace3fb8f9/Automation%20UseCase/Workflow%20Gsheet%20output.png)
