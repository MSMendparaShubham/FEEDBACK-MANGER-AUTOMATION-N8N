# AI Event Feedback Analyzer (n8n)

An n8n workflow that automatically analyzes event feedback submitted via a Google Form, classifies it using Claude (Anthropic), and routes it to the right place — coordinator alerts, a testimonials sheet, or a negative-feedback log — without any manual reading of raw responses.

Built for CMPICA, CHARUSAT University event feedback forms, but the logic is generic enough to reuse for any Google Forms → Google Sheets feedback pipeline.

**Testing:** This workflow has been tested end-to-end on real-time data from the Cyber Kavach event — 213 live student feedback responses were processed through it.

## What it does

1. **Trigger** — Watches a Google Sheet (`New Feedback Row`) for new rows, polling every minute. The sheet is the standard "Form responses 1" tab a Google Form writes to.
2. **Analyze (LLM)** — Sends each response to Claude Sonnet 4.6 with a carefully written analyst prompt. Claude returns strict structured JSON (via a schema-constrained output parser) with:
   - `sentiment` — Positive / Negative / Neutral
   - `emotion` — single dominant emotion
   - `key_insight` — one-sentence takeaway for the coordinator
   - `priority` — Critical / High / Medium / Low
   - `recommendation` — one concrete next step
   - `testimonial` — a short, natural quote (only if sentiment is Positive)

   The prompt explicitly tells the model to weigh written sentiment over raw star ratings (since students often mis-score), ignore blank fields instead of penalizing them, and avoid inventing details.

3. **Combine** — A Code node merges the original form fields (name, department, ratings, comments) with the LLM's structured output into one flat object.

4. **Route** — Three parallel filters act on every response:
   - **Not Low priority** → sends a formatted alert email to the coordinator with sentiment, emotion, priority, key insight, and recommendation.
   - **Positive** → appends the response (plus testimonial) to a Testimonials Sheet, and batches all positive reviews from the run into a single summary email.
   - **Negative** → appends the full response to a Coordinator Sheet for follow-up/tracking.

## Workflow diagram

```
Google Sheets Trigger (new row)
        │
        ▼
Analyze Feedback (Claude Sonnet 4.6 + structured schema)
        │
        ▼
  Combine Analysis (Code)
        │
   ┌────┼────────────────┐
   ▼    ▼                ▼
Not Low  Positive      Negative
Priority   │              │
   │       ├─ Append to    └─ Append to
   ▼       │  Testimonials    Coordinator Sheet
Send Alert │  Sheet
Email      └─ Build + Send
           Positive Summary
           Email
```

## Setup

### Prerequisites
- An n8n instance (cloud or self-hosted)
- A Google Form collecting event feedback, feeding into a Google Sheet
- Google Sheets OAuth2 credentials (trigger + write access)
- Gmail OAuth2 credentials
- An Anthropic API key (for the Claude Chat Model node)

### Steps
1. Import `workflow.json` into n8n.
2. Reconnect credentials on each node (Google Sheets Trigger, Google Sheets writes, Gmail, Anthropic Chat Model) — credentials are not portable between n8n instances.
3. Update the **Google Sheets Trigger** node to point at your form's response sheet.
4. Update the field names inside the **Analyze Feedback** prompt and the **Combine Analysis** code node if your form's question wording differs — they're currently matched to specific column headers.
5. Update the destination sheet IDs in **Append Good Review** and **Append Bad Review**, and the recipient email in both Gmail nodes.
6. Activate the workflow.

## Customization notes

- **Priority thresholds** — the "Actionable (not Low)" filter currently alerts on anything above Low priority. Tighten this to `Critical`/`High` only if alert volume gets noisy.
- **Model** — currently set to Claude Sonnet 4.6 at temperature 0.4 for consistent classification. Lower temperature further for more deterministic labeling.
- **Schema** — the structured output schema lives in the "Feedback Schema" node; extend it if you want additional fields (e.g. department-level tagging).

## Tech stack

- [n8n](https://n8n.io/) — workflow orchestration
- [Anthropic Claude](https://www.anthropic.com/) — feedback classification (LLM + structured JSON output)
- Google Sheets — data source and output logging
- Gmail — coordinator notifications
