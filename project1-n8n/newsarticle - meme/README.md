# News Article Meme Builder (n8n + Discord + Gemini + Gmail)

An n8n automation that turns news article URLs posted in a Discord channel into memes using Google Gemini (Nano Banana), then returns the meme to Discord and optionally emails it as a Gmail attachment. The workflow includes robust error handling and interactive user prompts.

- Input: News article URL in Discord
- Output: AI-generated meme image posted in Discord; optional Gmail attachment delivery
- Core stack: n8n, Discord Bot, Google Gemini (Nano Banana), Gmail

---

## Table of Contents
- [Overview](#overview)
- [Architecture](#architecture)
- [Features](#features)
- [Prerequisites](#prerequisites)
- [Setup & Configuration](#setup--configuration)
  - [1) n8n](#1-n8n)
  - [2) Discord](#2-discord)
  - [3) Google Gemini (Nano Banana)](#3-google-gemini-nano-banana)
  - [4) Gmail](#4-gmail)
  - [5) Import the workflow](#5-import-the-workflow)
  - [6) Configure node credentials and variables](#6-configure-node-credentials-and-variables)
- [Usage](#usage)
- [Technical Highlights](#technical-highlights)
  - [Base64 ➜ Binary conversion (n8n Function node)](#base64--binary-conversion-n8n-function-node)
- [Failure Handling](#failure-handling)
- [Troubleshooting](#troubleshooting)
- [Security Notes](#security-notes)
- [Project Files](#project-files)
- [Roadmap / Ideas](#roadmap--ideas)
- [License](#license)

---

## Overview
This project automates “AI meme-ification” of news article URLs shared in Discord. When a user posts a URL, n8n detects the latest message, sends the URL to an AI Agent node (Google Gemini Nano Banana) to generate a meme image, converts the resulting base64 to binary, and posts the meme back to Discord. The bot then prompts the user to send the meme via Gmail; on a “yes” reply, it emails the meme as an attachment. The workflow includes friendly error messages for AI/image generation or parsing failures.

---

## Architecture

```mermaid
flowchart TD
  A[User posts news URL in Discord channel] --> B[n8n: Discord trigger/get latest message]
  B --> C[AI Agent (Gemini Nano Banana): Create meme image from article]
  C -->|base64 image| D[Function: Convert base64 -> binary]
  D --> E[Discord: Post meme + ask 'Send via Gmail?']
  E --> F[Discord: Wait for user reply]
  F -->|yes| G[Gmail: Send email with meme attachment]
  F -->|no/other| H[Discord: Send emoji response]
  C -->|error| X[Discord: 'Sorry, I couldn’t create a meme...']
  D -->|error| Y[Discord: 'Error in parsing the code...']
  E -->|other errors| Z[Discord: 'Sorry, an error occurred...']
```

---

## Features
- News article ➜ meme workflow (“AI meme-ification” of URLs)
- Discord bot for automated message, file, and prompt handling
- Meme generation via Google Gemini’s Nano Banana model
- JavaScript base64-to-binary conversion for image attachment
- Optional Gmail attachment sending on user approval
- Predefined, user-friendly error and fallback messages

---

## Prerequisites
- n8n (self-hosted or cloud). Recommended: n8n v1.50+.
- Discord application/bot with:
  - Bot token
  - Permissions to read/send messages and upload files in target channel
- Google Gemini (Nano Banana) API access (Google AI Studio / Vertex AI or provider supporting “Nano Banana”)
- Google account with Gmail API enabled + OAuth 2.0 credentials
- A Discord server and channel where the bot is invited
- Node/network access for n8n to reach Discord, Google APIs, and Gmail

---

## Setup & Configuration

### 1) n8n
- Install n8n locally (Docker, npm, or desktop) or use n8n Cloud.
- Ensure you can access the n8n editor UI.
- Create required Credentials (Discord, Google Gemini, Gmail) in n8n’s Credentials store.

### 2) Discord
1. Create a Discord Application and add a Bot in the [Discord Developer Portal](https://discord.com/developers/applications).
2. Copy the Bot Token and store it in n8n as a Discord credential.
3. Invite the bot to your server with permissions:
   - Read Messages/View Channel
   - Send Messages
   - Attach Files
   - Read Message History
4. Get the target Channel ID (Developer Mode ➜ right-click channel ➜ Copy ID) or set up a webhook if using webhook-based nodes.

### 3) Google Gemini (Nano Banana)
- Obtain API credentials from Google AI Studio (or the platform that exposes Gemini “Nano Banana”).
- In n8n, configure an AI/LLM credential to use Gemini and select or input the “Nano Banana” model.
- If your provider requires a model name string, keep it consistent with the AI Agent node configuration.

### 4) Gmail
1. In Google Cloud Console, enable the Gmail API.
2. Create OAuth 2.0 Client credentials; add the n8n OAuth redirect URI (as shown in n8n Credentials UI).
3. In n8n, create a Gmail OAuth2 credential and complete authorization flow.
4. Ensure the Gmail sender has permission to send mail and that the recipient(s) are allowed.

### 5) Import the workflow
- Import the provided JSON into n8n:
  - In n8n UI ➜ Workflows ➜ Import ➜ choose `Meme-Builder.json`.

### 6) Configure node credentials and variables
- Open the imported workflow and configure:
  - Discord nodes: Select your Discord credential; set Channel ID or webhook.
  - AI Agent node: Select Gemini credential and “Nano Banana” model.
  - Function/Code node: Ensure it reads the base64 from the AI node output field.
  - Gmail node: Select your Gmail credential; set To/Subject/Body fields.
- Optional environment-style mapping (example):
  ```bash
  # Example: values you’ll map into n8n nodes/credentials
  DISCORD_CHANNEL_ID=123456789012345678
  DISCORD_BOT_TOKEN=xxxxxxxxxxxxxxxxxxxxxxxx
  GEMINI_API_KEY=ya29.xxxxxx
  GEMINI_MODEL="gemini-nano-banana"
  GMAIL_TO="you@example.com"
  GMAIL_FROM="bot@yourdomain.com"
  ```

---

## Usage
1. Post a news article URL in the configured Discord channel.
2. The bot generates a meme via Gemini and posts the image back to the channel.
3. The bot asks: “Send as Gmail attachment? Reply ‘yes’ to send.”
4. Reply “yes” to trigger the Gmail send; any other reply returns a playful emoji response.
5. If meme/image generation fails, an error message is returned in Discord.

Notes:
- The workflow checks the latest channel message as the URL input.
- All major branches and errors return predefined responses.

---

## Technical Highlights

- Latest message detection in the Discord channel via n8n nodes
- Dynamic prompt engineering for meme creation with Gemini (Nano Banana)
- Base64 ➜ binary conversion to attach images in Discord and Gmail
- Discord message interactions for approval flow
- IF/branching logic on user reply: “yes” vs. other
- Gmail attachment emailing with conditional execution paths

### Base64 ➜ Binary conversion (n8n Function node)
Use a Function (or Code) node to convert the AI’s base64 output into a binary file for downstream nodes.

Example Function node code:
```javascript
// Expects the previous node to output { memeBase64: "<base64 string>" }
// Optionally strip a data URL prefix if present.
const raw = $json.memeBase64 || '';
const base64 = raw.replace(/^data:image\/\w+;base64,/, '');

const buffer = Buffer.from(base64, 'base64');

// Prepare binary to attach; the property name "meme" is arbitrary but must match downstream nodes.
const binaryData = await this.helpers.prepareBinaryData(buffer, 'meme.png', 'image/png');

return [
  {
    json: { fileName: 'meme.png' },
    binary: { meme: binaryData },
  },
];
```

Discord/Gmail node configuration:
- Binary Property: `meme`
- File Name: use `{{$json.fileName}}` or set explicitly to `meme.png`

---

## Failure Handling
Predefined friendly messages are sent to Discord on failures:
- Meme generation fails:
  - “Sorry, I couldn’t create a meme for that link. Please try another article.”
- Code parsing/base64 conversion fails:
  - “Error in parsing the code, please try again by resending the link.”
- Other generic errors:
  - “Sorry, an error occurred, please try again.”

Tips:
- Consider enabling “Continue On Fail” selectively and route errors to a consolidated error handler node.
- Optionally add a global “Error Workflow” in n8n for incidents/alerts.

---

## Troubleshooting
- Bot not responding:
  - Verify bot permissions and Channel ID; ensure the trigger node or fetch-latest-message node is correctly scoped.
  - Check n8n workflow is active (if trigger-based).
- No image or broken file:
  - Ensure base64 parsing strips any data URL prefix.
  - Verify AI response field name matches the Function node’s input (`memeBase64`).
- Gmail fails to send:
  - Re-authenticate Gmail OAuth credential.
  - Check attachment “Binary Property” matches the Function node output (`meme`).
- Rate limits:
  - Discord and Google APIs enforce limits; add throttling/retry as needed.

---

## Security Notes
- Never commit raw API keys or OAuth secrets to the repo.
- Use n8n Credentials store for secrets; restrict access to your n8n instance.
- Limit Discord bot permissions to only what’s required.
- For Gmail, consider domain-level restrictions and sender policies (SPF/DKIM/DMARC) when using custom domains.

---

## Project Files
- [Meme-Builder.json](./Meme-Builder.json) — Complete n8n workflow (import into n8n)

If you add demo assets, logs, or recordings, include them here:
- links/ — screenshots, proof videos, and other evidence (optional)

---

## Roadmap / Ideas
- Support multiple model providers and automatic fallback
- Allow user-provided meme style (e.g., “sarcastic,” “wholesome,” “minimal”)
- Add content moderation for URLs
- Caching to avoid regenerating memes for the same link
- Persistent logs/analytics of meme requests and outcomes

---

## License
This project is provided as-is. If you intend to publish or distribute, consider adding a LICENSE file (e.g., MIT or Apache-2.0).