# Blaze Speed-to-Lead (Option B)

A four-node n8n pipeline that turns a ~22-hour lead response time into under 60 seconds.

A fake Facebook lead-ad submission hits a webhook → the lead is appended to a Google Sheet → an LLM scores fit (1–5) with a one-line reason and drafts a personalized first reply using structured output → the result is posted to Discord.

```
POST /webhook/lead  →  Google Sheets (append row)  →  OpenAI (gpt-4o-mini, JSON schema)  →  Code (flatten)  →  Discord
```

## What's in this repo

| File | What it is |
|---|---|
| `speed-to-lead.json` | Exported n8n workflow. Import it into any n8n instance. Credentials are referenced by name only — no secrets inside. |
| `prompt.md` | The system prompt and JSON schema used for scoring + drafting. |
| `sample_payload.json` | A simulated Facebook lead-ad payload for testing. |

## How to run it

1. **Import** `speed-to-lead.json` into n8n (Workflows → Import from file).
2. **Add credentials** (n8n will prompt for each node):
   - Google Sheets — sign in with Google; create a sheet with headers `name, email, phone, company, doors, message, received_at`.
   - OpenAI — API key (n8n cloud trial includes free gateway credits).
   - Discord — a channel webhook URL (Edit Channel → Integrations → Webhooks).
3. **Publish** the workflow so the production webhook URL is live.
4. **Send a test lead** (replace the host with yours):

```bash
curl -X POST "https://<your-instance>.app.n8n.cloud/webhook/lead" \
  -H "Content-Type: application/json" \
  -d @sample_payload.json
```

5. Watch Discord. Compare the `received_at` timestamp in the sheet to the Discord message time — end to end runs in a few seconds.

## Sample output

Input (`sample_payload.json`):

```json
{
  "name": "Jordan Lee",
  "email": "jordan@summitpm.com",
  "phone": "305-555-0142",
  "company": "Summit Property Management",
  "doors": 180,
  "message": "We manage 180 units and bookkeeping is a mess, late rent tracking especially."
}
```

LLM structured output:

```json
{
  "fit_score": 5,
  "reason": "The explicit mention of bookkeeping being a mess and late rent tracking indicates strong pain.",
  "reply": "Hi Jordan, I noticed you're managing 180 units and mentioned that your bookkeeping is a mess, particularly with tracking late rent. I'd love to explore how our services can help you streamline this process—would you be open to a quick 15-minute call?"
}
```

Discord message:

> 🔔 New lead — fit 5/5
> **Jordan Lee** · Summit Property Management · 180 doors · jordan@summitpm.com
> Why: The explicit mention of bookkeeping being a mess and late rent tracking indicates strong pain.
>
> Draft reply:
> Hi Jordan, I noticed you manage 180 units and mentioned that bookkeeping is a mess, particularly with late rent tracking. It sounds like you could benefit from streamlined processes—would you be open to a quick 15-minute call to discuss how we can help?

## Design notes

- **Structured output, not "please return JSON."** The OpenAI node uses a strict JSON schema (`fit_score`, `reason`, `reply`, no additional properties), so the downstream nodes can't break on a malformed answer.
- **Temperature 0 for scoring.** During testing the same lead scored 4 then 5 on consecutive runs. Scoring should be repeatable; creativity belongs in the reply text, so temperature is pinned to 0.
- **Signals in the rubric come from the lead itself** (door count, pain language, software/portal mentions, HOA/commercial footprint, growth language). Signals that need external enrichment (job postings, review mining, BBB complaints) are deliberately out of scope for a 2-hour build — see "With more time."
- **A Code node flattens the LLM response.** The first run returned the JSON as an escaped string inside a `message` field because the schema box still had n8n's placeholder. Fixed the schema, and kept a small parse/flatten step so the Discord node reads clean top-level fields.

## With more time

- Enrich each lead before scoring: hiring signals (bookkeeper / office manager postings), stated doors on the website, rent-portal vendor detection (AppFolio, Buildium, Yardi), late-fee language, review-text keyword hits, BBB complaint counts.
- Route by score: 4–5 → SMS to a rep plus calendar link; 1–2 → nurture sequence.
- Log the LLM's score and reason back to the sheet, then calibrate the rubric against real won/lost outcomes.
- Add retries and an error-notification branch so a failed LLM call never silently drops a lead.
