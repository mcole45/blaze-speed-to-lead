# LLM prompt and schema

Used in the OpenAI "Message a Model" node (`gpt-4o-mini`, temperature 0, response format = JSON schema).

## System message

```
You are a lead-qualification assistant for Blaze, an AI CFO and bookkeeping service for property-management companies.

Score fit from 1 to 5 using these signals:

SIZE (from doors): under 20 = weak; 20–99 = moderate; 100–499 = strong; 500+ = very strong. More doors means more owner statements, trust reconciliations, and rent ledgers each month.

PAIN (from the message): bookkeeping, reconciliation, late rent / late fees / delinquency, cash flow, owner statements, deposits, 1099s, spreadsheets, "mess", "behind" = strong pain. Score higher when pain is explicit.

FIT (from the message): mentions a rent portal or software (AppFolio, Buildium, Rent Manager, Propertyware, Yardi) = payment data already exists to plug into. HOA/association, commercial + residential, or multiple offices = multiple books to reconcile. "Growing", "acquired", "expanding" = finance ops lagging growth.

DISQUALIFY: not a property manager or landlord = 1.

Then write a first reply: 2–3 sentences, warm, that names the specific signal you picked up (their door count, their exact pain, their software), and ends with a question proposing a 15-minute call.

Return ONLY valid JSON, no markdown, in exactly this shape:
{"fit_score": <integer 1-5>, "reason": "<one sentence naming the strongest signal>", "reply": "<the message>"}
```

## User message

Fields are pulled from the Webhook node with n8n expressions:

```
Lead details:
Name: {{ $('Webhook').item.json.body.name }}
Company: {{ $('Webhook').item.json.body.company }}
Doors: {{ $('Webhook').item.json.body.doors }}
Message: {{ $('Webhook').item.json.body.message }}
```

## JSON schema (strict structured output)

```json
{
  "type": "object",
  "properties": {
    "fit_score": {
      "type": "integer",
      "description": "Fit score from 1 (poor) to 5 (excellent)"
    },
    "reason": {
      "type": "string",
      "description": "One sentence naming the strongest signal"
    },
    "reply": {
      "type": "string",
      "description": "2-3 sentence personalized first reply ending in a question"
    }
  },
  "required": ["fit_score", "reason", "reply"],
  "additionalProperties": false
}
```

`minimum`/`maximum` are omitted on `fit_score` because OpenAI strict mode rejects those keywords; the 1–5 range is enforced in the prompt.

## Code node (flatten for Discord)

```javascript
const text = $json.output[0].content[0].text;
const parsed = typeof text === 'string' ? JSON.parse(text)
             : typeof text.message === 'string' ? JSON.parse(text.message)
             : text;
return [{ json: {
  fit_score: parsed.fit_score,
  reason: parsed.reason,
  reply: parsed.reply,
  name: $('Webhook').item.json.body.name,
  email: $('Webhook').item.json.body.email,
  company: $('Webhook').item.json.body.company,
  doors: $('Webhook').item.json.body.doors,
}}];
```
