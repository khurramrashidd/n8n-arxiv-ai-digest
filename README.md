# Daily AI/ML Papers Digest (n8n Workflow)

An automated n8n workflow that fetches the latest AI/ML research papers from arXiv every day, summarizes each abstract using Google Gemini, and emails a clean digest straight to your inbox — no manual scanning of arXiv required.

## What it does

Every morning at 8 AM, the workflow:

1. Fetches the 5 most recent papers from arXiv matching `cat:cs.AI OR cat:cs.LG`
2. Parses the XML response into structured paper data (title, abstract, link, published date)
3. Loops through each paper and generates a 3-sentence, non-expert-friendly summary using Google Gemini
4. Combines each paper's title, link, and AI-generated summary into a single record
5. Formats everything into a readable email digest
6. Sends it via Gmail

## Architecture

```
Schedule Trigger (8 AM)
        ↓
Fetch arXiv Papers (HTTP Request → arXiv API)
        ↓
Parse XML Response (Code node — custom regex parser)
        ↓
Loop Over Papers
        ↓
Summarize Abstract (Google Gemini Chat Model)
        ↓
Combine Paper Data (Set node — merges title/link with AI summary)
        ↓
Create Email Digest (formats final HTML/text digest)
        ↓
Send Gmail Digest
```

## Problems solved while building this

This wasn't a one-shot generation — a few real issues came up that needed debugging:

- **`xml2js` module blocked on n8n Cloud**: The auto-generated XML parsing node tried to `require('xml2js')`, which n8n Cloud's sandboxed Code node environment disallows. Fixed by rewriting the parser using plain regex string matching instead of an external library.

- **AI node dropping fields**: The Gemini summarization node (a LangChain-based chat node) only returns its own `output` field by default and discards all other input data — so `title` and `link` were getting lost by the time the data reached the email step. The initial attempt to fix this with a `Merge` node (Combine by Position) failed unreliably because position-based merging doesn't work cleanly across a Loop node boundary. The working fix: removing the Merge node entirely and having the "Combine Paper Data" node reference the original "Loop Over Papers" output and the "Summarize Abstract" output directly by node name, rather than relying on item position.

- **API key suspension**: An initial Gemini API key got suspended by Google; resolved by generating a fresh key through Google AI Studio under a clean project.

- **Gmail OAuth scope**: Used the minimum required Gmail scope ("Read, compose, and send emails") rather than full read/delete access, since the workflow only needs to send mail.

## Setup

### Prerequisites
- An [n8n](https://n8n.io) account (cloud or self-hosted)
- A free Gemini API key from [Google AI Studio](https://aistudio.google.com/apikey)
- A Gmail account for sending the digest

### Steps
1. Import `daily-ai-ml-papers-digest.json` into your n8n instance
2. Set up credentials:
   - **Google Gemini (PaLM) API**: paste your AI Studio API key
   - **Gmail OAuth2**: connect your Google account, granting only the "compose and send" scope
3. Open the **Send Gmail Digest** node and set the `To` field to your email address
4. (Optional) Adjust the **Schedule Trigger** time or the arXiv `search_query` parameter to target a different topic
5. Run once manually to test, then activate the workflow

## Tech used

- n8n (workflow orchestration)
- arXiv public API
- Google Gemini (LLM summarization)
- Gmail API (delivery)

## Notes

- arXiv's API is free and requires no authentication.
- Google AI Studio's free tier has rate limits (a handful of requests per minute) — if you see "Service unavailable" errors mid-run, enabling **Retry On Fail** on the Summarize Abstract node helps smooth over transient rate-limit hits.
