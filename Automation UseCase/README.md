# Review Agent Automation (WhatsApp + Google Sheets + Custom GPT)

Automate end-to-end product review analysis, enrichment, and distribution from an e-commerce URL. This n8n workflow ingests a product URL, scrapes reviews and product details, combines signals, routes queries through a custom GPT, generates a buying-guide style summary in Markdown, and delivers output to Google Sheets and WhatsApp.

- Three execution paths:
  - WhatsApp-triggered insights for ad-hoc analysis.
  - Google Sheets-triggered runs for batch processing.
  - Custom GPT webhook for external apps and the flagship conversational experience.
- Sources: Amazon product pages and reviews (via Apify Actor) with competitor enrichment through Perplexity.
- Outputs: WhatsApp summary + Google Sheet append/update + Custom GPT response payloads.
- Models: Azure OpenAI, Google Gemini, and a Custom GPT profile orchestrated via LangChain Model Selector.

> Workflow JSON: [Automation UseCase/Review Agent.json](https://github.com/nxtrohith/abcd-agentic-training-vnr-nxtrohith/blob/main/Automation%20UseCase/Review%20Agent.json)

---

## Visuals

![Workflow Snapshot](https://github.com/nxtrohith/abcd-agentic-training-vnr-nxtrohith/blob/fb78cb4a1f9902f1f6e4e31f2b478b0ace3fb8f9/Automation%20UseCase/Workflow%20Snapshot.png)

WhatsApp Output:
![WhatsApp Output](https://github.com/nxtrohith/abcd-agentic-training-vnr-nxtrohith/blob/fb78cb4a1f9902f1f6e4e31f2b478b0ace3fb8f9/Automation%20UseCase/Workflow%20WhatsApp%20output.png)

Google Sheets Output:
![GSheet Output](https://github.com/nxtrohith/abcd-agentic-training-vnr-nxtrohith/blob/fb78cb4a1f9902f1f6e4e31f2b478b0ace3fb8f9/Automation%20UseCase/Workflow%20Gsheet%20output.png)

Custom GPT Website:
![Custom GPT Website](./Review%20Agent%20Website.png)

Custom GPT Response:
![Custom GPT Response](./CustomGPT%20Response.png)

Demo videos:
- WhatsApp run: [Worflow WhatsApp Execution.mp4](https://github.com/nxtrohith/abcd-agentic-training-vnr-nxtrohith/blob/fb78cb4a1f9902f1f6e4e31f2b478b0ace3fb8f9/Automation%20UseCase/Worflow%20WhatsApp%20Execution.mp4)
- Google Sheets run: [Workflow GSheets Execution.mp4](https://github.com/nxtrohith/abcd-agentic-training-vnr-nxtrohith/blob/fb78cb4a1f9902f1f6e4e31f2b478b0ace3fb8f9/Automation%20UseCase/Workflow%20GSheets%20Execution.mp4)

---

## Custom GPT Flagship Feature

- A dedicated Custom GPT front-end (web and shareable link) that chatifies review generation.
- Seamless reuse of existing summaries via Google Sheet lookups before launching new scrapes.
- Dynamic orchestration of Azure OpenAI, Google Gemini, and Perplexity within the same conversation.
- Instant webhook responses that mirror the WhatsApp Markdown layout for consistency.
- Visual experience captured above (website + response screenshots) for quick onboarding.

---

## High-Level Flow

Fetch Product Data → Collect Reviews → Process & Combine → Custom GPT Reasoning → Append/Update Google Sheet → Send Responses (WhatsApp or Webhook)

Three orchestrated paths:

- WhatsApp URL → Sheet lookup via Custom GPT → (if missing) Apify + HTTP scrape → Combine Reviews → Reviewer agent → Google Sheets append → WhatsApp reply.
- Google Sheet trigger → Filter blank outputs → Scrape product + reviews → Combine reviews → Reviewer agent → Google Sheets append/update → Optional Perplexity competitor lookup.
- Custom GPT webhook → Sheet lookup (reuse summaries when present) → On-demand scrape + reviewer agent + Perplexity enrichment → JSON response for downstream apps.

---

## Architecture

- n8n workflow with:
  - WhatsApp Trigger + Send nodes (Cloud API).
  - Google Sheets Trigger + Tool nodes for append and update against the same Review Request Sheet (GID 1238187015).
  - Webhook endpoint powering the Custom GPT experience.
  - AI Agent nodes that run sheet lookups before any heavy scrape (dedupe pathway).
  - Apify Actor (Amazon Reviews Scraper) capped at `maxReviews: 100`.
  - HTTP Request nodes for product HTML fetches with full browser headers.
  - Code (JavaScript) nodes for parsing characteristics and combining reviews.
  - LangChain Model Selector driving Azure OpenAI, Google Gemini, and custom GPT prompts.
  - Perplexity tool nodes that enrich summaries with competitor intelligence when reviews lack it.

ASCII overview:

```
[WhatsApp Trigger]
  ├─► [AI Agent2 Sheet Lookup] ──► (summary exists?) ──► yes ► [Send message1]
  │                                          └─► no ► [Apify Reviews] ─► [Merge]
  └─► [HTTP Product HTML] ─► [Parse Characteristics] ─┘           │
                                            ▼
                                    [Set] ─► [Combine Reviews] ─► [Reviewer] ─► [GSheets Append] ─► [WhatsApp Send]

[Google Sheets Trigger] ─► [Filter Blank Rows] ─► [Get Rows] ─► [HTTP HTML]
                              │
                              └─► [Apify Reviews] ─┐
                                            ▼
                                    [Merge1] ─► [Split Out] ─► [Set1] ─► [Combine Reviews1]
                                            │
                                            ▼
                                    [Reviewer1 + Perplexity] ─► [GSheets Append/Update] ─► [Loop]

[Webhook1 Custom GPT] ─► [AI Agent1 Sheet Lookup]
          │
          └─► (summary missing?) ─► [Apify Reviews2] + [HTTP Product HTML2] ─► [Combine Reviews2] ─► [Reviewer1]
                        │                                          │
                        └──────────────────────────────────────────┘
                                 ▼
                              [Respond to Webhook]
```

---

## Nodes and Responsibilities

### Branch 1 — WhatsApp-Triggered Review Summary

1) WhatsApp Trigger
- Listens for incoming messages containing a product URL (`$json.messages[0].text.body`).
- Immediately fans out to the sheet lookup agent and the product HTML fetch.

2) AI Agent2 (Custom GPT Sheet Lookup)
- Searches the Review Request Sheet (GID 1238187015) for an existing summary.
- If found, the response is echoed back to WhatsApp without re-scraping.
- If no match exists, the pipeline proceeds to scraping.

3) Review Scraper (Apify Actor: Amazon Reviews Scraper)
- Parameters: `maxReviews=100`, `filterByRatings=allStars`, `productUrls=[WhatsApp URL]`.
- On failure, the workflow falls back to a WhatsApp alert and exits.

4) Product Page Scraper (HTTP Request)
- Fetches raw HTML with browser-like headers; TLS validation relaxed for resilience.

5) Characteristics Fetcher (Code)
- Parses HTML to extract title, description, and specifications.
- Normalizes to a JSON payload containing `title`, `description`, `specifications[]`, and the canonical `input` URL.

6) Merge
- Combines Apify reviews with parsed product details using `input` as the join key.

7) Sets info (Set)
- Normalizes fields such as `reviewDescription`, `stars`, `Product Descreiption`, `title`, and `URL` for downstream nodes.

8) Combines Reviews (Code)
- Aggregates reviews, counts entries, and computes the average star rating.
- Produces a Markdown-ready `combinedReviews` block separated by `---` for clarity.

9) Reviewer (LangChain Agent + Custom GPT instructions)
- Uses LangChain Model Selector to route requests to Azure OpenAI, Google Gemini, or the Custom GPT persona.
- Generates a buying-guide format with overview, ideal buyers, pros/cons, worth noting, comparisons, and a verdict.
- Triggers a Perplexity competitor lookup when the prompt requests external comparisons.

10) Google Sheets Tool (Append)
- Appends Product Title, URL, Product Description, Average Rating, and the review summary to Sheet1 (gid=0).

11) WhatsApp Send
- Sends the generated Markdown summary back to the originating user.

---

### Branch 2 — Google Sheet-Triggered Review Generator

1) Google Sheets Trigger
- Polls “Review Request Sheet” (GID 1238187015) every minute for changes in “Product Url”.
- Only emits rows where the watched column changed.

2) Filter
- Continues only if “Product Title” remains empty (indicates no prior run).

3) Get row(s) in sheet
- Retrieves the first row whose “Output” column is blank, enabling deterministic batching.

4) Product Page Scraper1 (HTTP)
- Fetches raw HTML with the same browser headers as Branch 1.

5) Review Scraper1 (Apify)
- Parameters mirror Branch 1 (`maxReviews=100`, `filterByRatings=allStars`).

6) Characteristics Fetcher1 (Code)
- Produces normalized metadata identical to Branch 1.

7) Merge1 and Split Out
- Merges product details with reviews, then splits into manageable batches for transformation.

8) Sets info1 + Combines Reviews1 (Code)
- Standardizes fields and produces aggregate metrics/combined review text.

9) Reviewer1 (LangChain Agent + Perplexity tool)
- Uses the Custom GPT prompt that mandates Markdown with comparisons.
- Calls the Perplexity tool node when the prompt requests competitor evidence.

10) Google Sheets Tool (Append + Update)
- Appends structured rows (Product Title, Product Url, Review Description, Output) back to the Review Request Sheet for auditing.
- A dedicated update node keeps the same row in sync when reprocessing.

11) Loop Over Items
- `SplitInBatches` iterates through multiple queued rows without re-triggering the workflow.

12) Error Handling
- On scraping errors, the WhatsApp notify path alerts `Error running node 'Review Scraper'`.

### Branch 3 — Custom GPT Webhook + Flagship Experience

1) Webhook1 endpoint
- Accepts POST payloads from the Custom GPT front-end and partner integrations.
- Captures `productName` and/or `url` and hands them to AI Agent1 for dedupe checks.

2) AI Agent1 (Sheet Lookup)
- Runs the custom GPT prompt to search existing Google Sheet rows.
- Returns cached Markdown summaries immediately when a match is found.

3) If branch (false path)
- When no summary exists, the workflow invokes Apify/HTTP nodes (`Review Scraper2`, `Product Page Scraper2`, `Characteristics Fetcher2`).
- The processing stack mirrors Branch 2 to guarantee parity across channels.

4) Reviewer1 + Perplexity
- Reuses the flagship Custom GPT prompt with comparison requirements and optional Perplexity competitor fetches.
- Persists the outcome back to Google Sheets for future reuse.

5) Respond to Webhook
- Returns a JSON payload tailor-made for the Custom GPT UI, enabling instant conversational replies.
- Sends follow-up notifications via WhatsApp when the webhook call originates from that channel.

---

## AI Model Selection

- Model Selector routes between:
  - Azure OpenAI (model-router backed, e.g., o4-mini variants).
  - Google Gemini (via PaLM/Gemini API credential).
  - Custom GPT persona that layers sheet lookups, Perplexity calls, and strict markdown formatting.
- Prompts are authored in LangChain Agent nodes with explicit instructions for competitor comparisons, dedupe policy, and sheet logging.
- Perplexity tool nodes are registered as AI tools, allowing the Custom GPT to request competitive context only when mandated.
- All branches share the same Markdown contract to keep WhatsApp, Sheets, and Custom GPT responses aligned.

---

## Example AI Output

```markdown
# Apple AirPods Pro (2nd Gen) — Should you buy it?

Overview: Owners praise the balanced audio, excellent active noise cancellation, and comfortable fit for long flights, though a few note battery life tapering after the first year. Set-up remains seamless inside the Apple ecosystem, with reviewers repeatedly highlighting how quickly the case recharges on a MagSafe pad.

Who it’s best for:
- Travelers who need quiet cabins
- iPhone users who swap devices often
- Commuters seeking compact ANC earbuds

Pros (from reviews):
- Elite ANC that blocks engines
- Secure fit once ear tips are swapped
- Spatial audio feels immersive
- Quick pairing across Apple gear
- Case supports wireless charging

Cons (from reviews):
- Battery longevity declines over time
- Pricey versus tuned rivals
- Some mics struggle in wind

Worth noting:
- Latest firmware stabilizes Bluetooth
- Foam tips help with smaller ears
- AppleCare+ covers battery swaps

How it compares to alternatives:
- Sony WF-1000XM5: similar on ANC, Apple leads on device handoff
- Bose QuietComfort Ultra: Bose is better on comfort, weaker on spatial audio
- Samsung Galaxy Buds2 Pro: better value for Android, not enough data to compare reliability

Verdict: Outstanding for Apple-first listeners who prioritise noise canceling and ecosystem perks over price.
```

---

## Setup

### Prerequisites
- n8n (self-hosted or cloud) with access to:
  - WhatsApp Cloud API credentials.
  - Google Sheets API OAuth (covering both trigger and write scopes).
  - Apify API token with access to actor “Amazon Reviews Scraper”.
  - Azure OpenAI and/or Google Gemini API keys (LangChain integrations).
  - Perplexity API key (for optional competitor enrichment).
- A Google Sheet (document ID `1jY-8MkxAzytOF_My-Cfg8DwDH4yrNGY_mcQbQFAdR6k`) with:
  - `Review Request Sheet` (gid `1238187015`) holding Product Url, Product Title, Review Description, Output.
  - `Sheet1` (gid `0`) for WhatsApp append logs (Product Title, Product Url, Product Description, Avg Rating, Review Description).
- Optional: Custom GPT front-end or shareable link that calls the `Webhook1` endpoint.

### Import the Workflow
- Download: [Review Agent.json](https://github.com/nxtrohith/abcd-agentic-training-vnr-nxtrohith/blob/main/Automation%20UseCase/Review%20Agent.json)
- In n8n, import from file and enable webhook URLs.
- Open the workflow and configure credentials on:
  - WhatsApp Trigger / WhatsApp Send.
  - Google Sheets Trigger / Google Sheets Tool.
  - Apify.
  - Azure OpenAI and/or Google Gemini.
  - Perplexity (optional but required for competitor comparisons).

### Configure IDs and Targets
- WhatsApp:
  - Phone Number ID and recipient strategy (dynamic routing or fixed QA number).
- Google Sheets:
  - Document URL/ID shared with the integration account.
  - `Sheet1` (gid `0`) for WhatsApp append logs.
  - `Review Request Sheet` (gid `1238187015`) for trigger, append, and update operations.
- Apify:
  - Actor: junglee/amazon-reviews-scraper (`R8WeJwLuzLZ6g4Bkk`).
  - Input configuration: `maxReviews=100`, `filterByRatings=["allStars"]`, `productUrls` set dynamically.
- AI:
  - Set fallback preferences in the LangChain Model Selector.
  - Update Custom GPT prompt text inside the agent nodes as needed.
- Perplexity:
  - Configure API key on both the tool and the chat node to allow comparison lookups.

---

## Key Code Patterns

HTML extraction (Characteristics Fetcher) – trimmed from the production node:
```js
return items.map((item) => {
  const payload = item.json;
  const html = typeof payload === 'string'
    ? payload
    : payload?.body ?? payload?.data ?? payload?.html ?? '';

  if (!html) {
    return {
      json: {
        title: null,
        error: 'No HTML string found from previous node',
        debugKeys: Object.keys(payload || {}),
      },
    };
  }

  const span = html.match(/<span id="productTitle"[^>]*>([\s\S]*?)<\/span>/i);
  let title = span ? span[1].replace(/\s+/g, ' ').trim() : null;

  if (!title) {
    const titleTag = html.match(/<title>([\s\S]*?)<\/title>/i);
    if (titleTag) {
      title = titleTag[1]
        .replace(/Amazon\.\w+:\s*/i, '')
        .replace(/\s+\|.*$/, '')
        .trim();
    }
  }

  if (!title && payload?.request?.url) {
    const slug = payload.request.url.split('/').filter(Boolean).pop();
    if (slug) title = slug.replace(/-/g, ' ').trim();
  }

  const descriptionBlock = html.match(/<div id="productDescription"[^>]*>([\s\S]*?)<\/div>/i)
    || html.match(/<div id="feature-bullets"[^>]*>([\s\S]*?)<\/div>/i)
    || html.match(/<div id="bookDescription_feature_div"[^>]*>([\s\S]*?)<\/div>/i);

  const description = descriptionBlock
    ? descriptionBlock[1].replace(/<[^>]+>/g, ' ').replace(/\s+/g, ' ').trim()
    : null;

  let specs = [];
  const specTable = html.match(/<table[^>]*id="productDetails_techSpec_section_1"[^>]*>([\s\S]*?)<\/table>/i);
  const detailBullets = html.match(/<div id="detailBullets_feature_div"[^>]*>([\s\S]*?)<\/div>/i);
  const rawSpec = specTable?.[1] ?? detailBullets?.[1] ?? '';

  if (rawSpec) {
    const rows = [...rawSpec.matchAll(/<tr[^>]*>([\s\S]*?)<\/tr>/gi)];
    if (rows.length) {
      specs = rows.map((row) => {
        const cells = [...row[1].matchAll(/<td[^>]*>([\s\S]*?)<\/td>/gi)];
        if (cells.length >= 2) {
          const key = cells[0][1].replace(/<[^>]+>/g, '').trim();
          const val = cells[1][1].replace(/<[^>]+>/g, '').trim();
          return `${key}: ${val}`;
        }
        return row[1].replace(/<[^>]+>/g, '').trim();
      });
    } else {
      specs = [...rawSpec.matchAll(/<li[^>]*>([\s\S]*?)<\/li>/gi)]
        .map((bullet) => bullet[1].replace(/<[^>]+>/g, '').trim());
    }
  }

  return {
    json: {
      title,
      description,
      specifications: specs,
      input: payload?.request?.url ?? item.json.input ?? item.json.url,
    },
  };
});
```

Review aggregation (Combines Reviews):
```js
const items = $input.all();
const reviews = items
  .map((item) => (item.json.reviewDescription || '').trim())
  .filter(Boolean);

const starValues = items
  .map((item) => Number(item.json.stars || 0))
  .filter((value) => !Number.isNaN(value));

const avgStars = starValues.length
  ? starValues.reduce((sum, value) => sum + value, 0) / starValues.length
  : null;

return [
  {
    json: {
      product: $input.first().json.title,
      totalReviews: reviews.length,
      avgStars,
      url: $('Characteristics Fetcher').first().json.input,
      ProductDescription: $input.first().json['Product Descreiption'],
      combinedReviews: reviews.join('\n\n---\n\n'),
    },
  },
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

- Custom GPT path:
  1) Hit the `Webhook1` endpoint via the Custom GPT UI or API with a product name/URL.
  2) Workflow checks the sheet cache, scrapes if needed, updates storage, and responds with Markdown suitable for display in chat.

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

- Custom GPT cache (Webhook branch):
  - Reuses Review Request Sheet (gid `1238187015`).
  - Stores latest `Review Description` and `Output` for quick lookups.

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
- Protect the Custom GPT webhook with auth (IP allowlist, signatures, or tokens) before exposing publicly.

---

## Extensibility

- Add other marketplaces by swapping the Apify actor and parse logic.
- Enhance deduplication and URL normalization pre‑scrape.
- Store raw datasets to a data warehouse (e.g., BigQuery, Postgres) for auditability.
- Add vector embedding for semantic retrieval over review corpora.
- Expand Custom GPT personas (e.g., brand-specific tone) by cloning the webhook branch.
- Swap Perplexity for an internal catalog service when competitor signals are available in-house.

---

## References

- Workflow JSON: [Automation UseCase/Review Agent.json](https://github.com/nxtrohith/abcd-agentic-training-vnr-nxtrohith/blob/main/Automation%20UseCase/Review%20Agent.json)
- WhatsApp demo: [Worflow WhatsApp Execution.mp4](https://github.com/nxtrohith/abcd-agentic-training-vnr-nxtrohith/blob/fb78cb4a1f9902f1f6e4e31f2b478b0ace3fb8f9/Automation%20UseCase/Worflow%20WhatsApp%20Execution.mp4)
- GSheets demo: [Workflow GSheets Execution.mp4](https://github.com/nxtrohith/abcd-agentic-training-vnr-nxtrohith/blob/fb78cb4a1f9902f1f6e4e31f2b478b0ace3fb8f9/Automation%20UseCase/Workflow%20GSheets%20Execution.mp4)
- Snapshots:
  - [Workflow Snapshot.png](https://github.com/nxtrohith/abcd-agentic-training-vnr-nxtrohith/blob/fb78cb4a1f9902f1f6e4e31f2b478b0ace3fb8f9/Automation%20UseCase/Workflow%20Snapshot.png)
  - [Workflow WhatsApp output.png](https://github.com/nxtrohith/abcd-agentic-training-vnr-nxtrohith/blob/fb78cb4a1f9902f1f6e4e31f2b478b0ace3fb8f9/Automation%20UseCase/Workflow%20WhatsApp%20output.png)
  - [Workflow Gsheet output.png](https://github.com/nxtrohith/abcd-agentic-training-vnr-nxtrohith/blob/fb78cb4a1f9902f1f6e4e31f2b478b0ace3fb8f9/Automation%20UseCase/Workflow%20Gsheet%20output.png)
