# WhatsApp Agent (n8n + WhatsApp Cloud API + Google Sheets + LLM)

Production-ready n8n automation that turns a WhatsApp number into a smart, tool-enabled assistant. It can chat naturally, remember context, and perform Google Sheets operations (read, append/update, delete, clear) to manage stock, orders, and chat logs.

- Workflow file: [WhatsApp Agent.json](https://github.com/nxtrohith/abcd-agentic-training-vnr-nxtrohith/blob/ebcb8bb56d41af146b7d25c442da0c248d2b6a84/project1-n8n/WhatsApp%20Agent/WhatsApp%20Agent.json)
- Snapshot: ![Workflow Snapshot](https://github.com/nxtrohith/abcd-agentic-training-vnr-nxtrohith/blob/ebcb8bb56d41af146b7d25c442da0c248d2b6a84/project1-n8n/WhatsApp%20Agent/Workflow%20Snapshot.png)
- Demo reply: ![Whatsapp Reply](https://github.com/nxtrohith/abcd-agentic-training-vnr-nxtrohith/blob/ebcb8bb56d41af146b7d25c442da0c248d2b6a84/project1-n8n/WhatsApp%20Agent/Whatsapp%20Reply.png)
- Sheets: 
  - Stock List: ![Google Sheets Stock List](https://github.com/nxtrohith/abcd-agentic-training-vnr-nxtrohith/blob/ebcb8bb56d41af146b7d25c442da0c248d2b6a84/project1-n8n/WhatsApp%20Agent/Google%20Sheets%20Stock%20List.png)
  - Orders: ![Google sheets Orders Page](https://github.com/nxtrohith/abcd-agentic-training-vnr-nxtrohith/blob/ebcb8bb56d41af146b7d25c442da0c248d2b6a84/project1-n8n/WhatsApp%20Agent/Google%20sheets%20Orders%20Page.png)
  - Chat DB: ![Google sheets Chat DB](https://github.com/nxtrohith/abcd-agentic-training-vnr-nxtrohith/blob/ebcb8bb56d41af146b7d25c442da0c248d2b6a84/project1-n8n/WhatsApp%20Agent/Google%20sheets%20Chat%20DB.png)
- Video: [Workflow Execution.mp4](https://github.com/nxtrohith/abcd-agentic-training-vnr-nxtrohith/blob/ebcb8bb56d41af146b7d25c442da0c248d2b6a84/project1-n8n/WhatsApp%20Agent/Workflow%20Execution.mp4)

## What it does

- Receives WhatsApp messages via Meta’s WhatsApp Cloud API.
- Uses an LLM (Google Gemini via n8n’s LangChain nodes) as a tool-using Agent.
- Persists chat context per user (Buffer Window memory).
- Reads/writes to Google Sheets for:
  - Stock list lookup and maintenance.
  - Order creation and updates (append-or-update by PID).
  - Chat logging (timestamped).
  - Admin actions like deleting rows or clearing ranges.

## Architecture

Core components in the workflow:

- WhatsApp Trigger (n8n-nodes-base.whatsAppTrigger)
- AI Agent (LangChain Agent) with tools and memory
  - Language model: Google Gemini Chat Model (@n8n/n8n-nodes-langchain.lmChatGoogleGemini)
  - Memory: Simple Memory (Buffer Window) keyed by WhatsApp identifiers
  - Tools: Google Sheets operations (read, append/update, update, delete, clear)
- WhatsApp Send Message (n8n-nodes-base.whatsApp)

High-level flow:
1. WhatsApp message hits the Webhook Trigger.
2. Memory constructs a session key from phone_number_id and wa_id.
3. LLM Agent processes the user query with a friendly, helpful system prompt.
4. Agent calls Google Sheets tools as needed (stock/order/chat DB).
5. Reply is sent back to the user on WhatsApp.
6. Chat is logged to “Chat DB” sheet.

## Data model (Google Sheets)

These sheets are referenced in the workflow:

- Stock List (gid=0, named “List”)
  - Columns: PID, Product, Availability, Cost
- Orders (gid=432305293)
  - Columns: Name, Date, PID, Product, Status
  - Append-or-update keyed by PID
- Chat DB (gid=731383016)
  - Columns: Timestamps, Phone id, Chat Body

You can use the same Spreadsheet ID with three separate tabs (recommended). Ensure the tab names and column headers match.

## Prerequisites

- n8n running locally, self-hosted, or n8n Cloud
- Meta (Facebook) developer account with WhatsApp Cloud API:
  - One WhatsApp Business Account
  - A phone number ID and access token
  - Webhook configured for message updates
- Google account with a Spreadsheet that contains the three tabs above
- Google API key/credentials for n8n’s Google Sheets and Google Gemini (PaLM/Gemini) nodes

## Setup

1) Clone the repo
```bash
git clone https://github.com/nxtrohith/abcd-agentic-training-vnr-nxtrohith.git
cd abcd-agentic-training-vnr-nxtrohith/project1-n8n/WhatsApp\ Agent
```

2) Start n8n (example: Docker)
```bash
docker run -it --rm \
  -p 5678:5678 \
  -v ~/.n8n:/home/node/.n8n \
  n8nio/n8n:latest
```
Or use n8n Cloud / your own deployment method.

3) Configure WhatsApp Cloud API
- In Meta developer portal, create or use an app with WhatsApp product.
- Set up a webhook callback URL pointing to your n8n instance (the WhatsApp Trigger node provides a test URL).
- Subscribe to “messages” updates.
- In n8n Credentials, add a WhatsApp credential with your access token (used by:
  - WhatsApp Trigger: credentials “WhatsApp OAuth account”
  - WhatsApp Send: credentials “WhatsApp account”)
- In the “Send message” node, set:
  - phoneNumberId to your own Phone Number ID
  - recipientPhoneNumber should typically be dynamic, not a hard-coded test number (replace with expression that uses the incoming wa_id when going live).

4) Configure Google services
- Create n8n credential “Google Sheets account” (OAuth2).
- Share the target Spreadsheet with the Google account used for the OAuth consent.
- Create n8n credential for “Google Gemini(PaLM) Api account” and add the Gemini API key.

5) Prepare your Spreadsheet
- Use one Spreadsheet with three tabs:
  - “List” (gid=0): PID, Product, Availability, Cost
  - “Orders” (gid=432305293): Name, Date, PID, Product, Status
  - “Chat DB” (gid=731383016): Timestamps, Phone id, Chat Body
- If you use different names/gids, update the Google Sheets nodes accordingly.

6) Import the workflow
- In n8n, Import from URL/File and select: [WhatsApp Agent.json](https://github.com/nxtrohith/abcd-agentic-training-vnr-nxtrohith/blob/ebcb8bb56d41af146b7d25c442da0c248d2b6a84/project1-n8n/WhatsApp%20Agent/WhatsApp%20Agent.json)
- Open each node and:
  - Bind your Credentials.
  - Replace hard-coded Spreadsheet URLs with your own Spreadsheet ID.
  - Replace any test phone numbers with expressions based on incoming data.

7) Activate the workflow
- Activate once webhook verification is complete and credentials are valid.
- Send a message to your WhatsApp number to test.

## Workflow internals

Nodes and responsibilities (IDs refer to the JSON for easy mapping):

- WhatsApp Trigger (id: 23c69687-…)
  - Subscribes to “messages” updates
  - Exposes webhook to accept WhatsApp events
- Simple Memory (id: d34dd3de-…)
  - Session key includes phone_number_id and wa_id to keep user-specific context
- Google Gemini Chat Model (id: 41d2b6fd-…)
  - Backing LLM for the Agent
- AI Agent (id: 70321d84-…)
  - System prompt: friendly, witty assistant; tool-enabled
  - Delegates to Google Sheets tools via $fromAI parameter mapping:
    - Get rows, Append/Update by PID, Update rows, Delete rows, Clear ranges
- Google Sheets tools
  - DB retrieval (Chat DB) (id: f1d7d814-…)
  - DB entry (append to Chat DB) (id: f3fa79db-…)
  - DB shredder (clear specific range in Chat DB) (id: f2708735-…)
  - Orders sheet helpers:
    - Read orders (id: a3982d3f-…)
    - Append or update orders (id: 173c96ed-…; matching on PID)
    - Delete rows from Orders (id: b0bc34d2-…)
  - Stock list update (id: 3ef92f57-…; matching on Product)
  - Generic read (id: c1dee422-…)
- Send message (id: 2a6e4ed9-…)
  - Sends AI output back to WhatsApp

## Typical interactions

The Agent infers intent from free-form user text and calls tools with structured parameters:
- “What’s the price of the iPhone 15?” → Read Stock List, respond with Cost and Availability.
- “Order Pixel 8 for me, PID PX-101” → Append/update Orders by PID with Status=Created.
- “Update price of AirPods to 9999” → Update Stock List row by Product.
- “Cancel orders 12 to 14” → Delete rows in Orders.
- “Clear last 50 chat logs” → Clear range in Chat DB.

Tip: Make sure your column names match exactly; the Agent’s $fromAI mappings rely on those semantic identifiers.

## Configuration details to update

After importing, review and adjust:
- WhatsApp Send node:
  - phoneNumberId: set to your number ID
  - recipientPhoneNumber: use an expression referencing the incoming contact ID (wa_id) instead of a fixed number for production
- All Google Sheets nodes:
  - documentId URL(s): replace with your Spreadsheet ID/URLs
  - sheetName: ensure correct tab IDs or names
- AI Agent prompt: tune tone/capabilities as needed

## Security and privacy

- Do not hardcode access tokens or phone numbers in the workflow; use n8n Credentials and environment variables.
- Restrict Spreadsheet sharing to the OAuth account used by n8n.
- Chat DB contains sensitive content (PII). Apply retention policies and access controls.

## Troubleshooting

- Webhook verification fails:
  - Ensure your n8n instance is reachable over HTTPS with a valid certificate.
  - Re-copy the exact webhook URL from the WhatsApp Trigger node.
- 401/403 on WhatsApp:
  - Token expired or missing scopes; reissue token and update Credentials.
- Google Sheets permission errors:
  - Share the Spreadsheet with the Google account used in OAuth Credentials.
  - Confirm correct Spreadsheet ID and gid.
- Agent not using tools:
  - Verify the AI Agent is wired to the tool nodes (ai_tool connections).
  - Check that the $fromAI fields map to your actual column names.
- Wrong recipient:
  - Replace any hard-coded recipient phone with the incoming wa_id.

## Project structure

- [WhatsApp Agent.json](https://github.com/nxtrohith/abcd-agentic-training-vnr-nxtrohith/blob/ebcb8bb56d41af146b7d25c442da0c248d2b6a84/project1-n8n/WhatsApp%20Agent/WhatsApp%20Agent.json) — n8n workflow
- Images and demo:
  - [Workflow Snapshot.png](https://github.com/nxtrohith/abcd-agentic-training-vnr-nxtrohith/blob/ebcb8bb56d41af146b7d25c442da0c248d2b6a84/project1-n8n/WhatsApp%20Agent/Workflow%20Snapshot.png)
  - [Whatsapp Reply.png](https://github.com/nxtrohith/abcd-agentic-training-vnr-nxtrohith/blob/ebcb8bb56d41af146b7d25c442da0c248d2b6a84/project1-n8n/WhatsApp%20Agent/Whatsapp%20Reply.png)
  - [Google Sheets Stock List.png](https://github.com/nxtrohith/abcd-agentic-training-vnr-nxtrohith/blob/ebcb8bb56d41af146b7d25c442da0c248d2b6a84/project1-n8n/WhatsApp%20Agent/Google%20Sheets%20Stock%20List.png)
  - [Google sheets Orders Page.png](https://github.com/nxtrohith/abcd-agentic-training-vnr-nxtrohith/blob/ebcb8bb56d41af146b7d25c442da0c248d2b6a84/project1-n8n/WhatsApp%20Agent/Google%20sheets%20Orders%20Page.png)
  - [Google sheets Chat DB.png](https://github.com/nxtrohith/abcd-agentic-training-vnr-nxtrohith/blob/ebcb8bb56d41af146b7d25c442da0c248d2b6a84/project1-n8n/WhatsApp%20Agent/Google%20sheets%20Chat%20DB.png)
  - [Workflow Execution.mp4](https://github.com/nxtrohith/abcd-agentic-training-vnr-nxtrohith/blob/ebcb8bb56d41af146b7d25c442da0c248d2b6a84/project1-n8n/WhatsApp%20Agent/Workflow%20Execution.mp4)

## License

This repository does not include a license file. Add one if you intend external usage or contributions.

## Credits

Built with:
- [n8n](https://n8n.io/)
- WhatsApp Cloud API
- Google Sheets API
- Google Gemini (via n8n LangChain nodes)
