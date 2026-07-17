# PetShop WhatsApp AI Assistant

A WhatsApp customer-service bot for a pet shop, built as a single [n8n](https://n8n.io) workflow. It handles greetings, price questions, scheduling, complaints, and image/audio messages — and is deliberately architected to keep LLM costs down, degrade gracefully under load, and never leave a customer without a reply.

## How it works

```
Webhook (Evolution API)
→ Rate Limiter: 10 req/10s per contact, truncates overly long messages
   → over the limit → polite "please wait" reply (never silent)
→ Media detection → image/audio described by Gemini, other file types get a static reply
→ Intent Classifier (Gemini 2.5 Flash Lite, no memory/tools): greeting | services | out_of_scope | exception
→ greeting / services / out_of_scope → instant canned reply, no further LLM call
→ exception → full AI Agent (Postgres memory + Google Sheets tools) handles scheduling,
   complaints, and anything that needs real reasoning or a spreadsheet lookup
```

## Why it's built this way

- **Cost-optimized by design.** Most incoming messages (greeting, price question, out-of-scope) never reach the expensive conversational agent — a cheap, fast classifier triages them first, and only genuinely ambiguous cases escalate to the full agent with memory and tools.
- **Fails loud, not silent.** Every LLM/HTTP call (classifier, media analysis, agent, outbound message) has retry-on-failure configured. Hitting the rate limit gets you a friendly reply, not dead air — the one thing worse than a slow bot is one that goes quiet.
- **Security-conscious.** Per-contact rate limiting, input length caps, and an explicit prompt-injection defense in the agent's system prompt (customer text is treated as data, never as instructions).
- **Session-aware without a database.** Lightweight per-contact state (already greeted? already asked prices?) lives in n8n's own persisted workflow storage — no Redis, no extra infra — and expires after 24h so it never goes stale.
- **Group-safe.** The connected WhatsApp instance ignores group chats, so multiple people in the same group can't bleed into one shared conversation context.

## Stack

- **n8n** — workflow engine (self-hosted, Docker)
- **Evolution API** — WhatsApp connector
- **OpenRouter** (Gemini 2.5 Flash Lite) — intent classification, media understanding, and the fallback conversational agent
- **Google Sheets** — price list lookup + human-handoff log
- **Postgres** — chat memory for the fallback agent

## Running locally

```bash
docker compose up -d
```

n8n will be available at `http://localhost:5678`. Import [`workflow.json`](workflow.json) into n8n and configure these credentials (create them in n8n's Credentials screen — no values are stored in this repo):

| Credential name       | Type                  | Used for                          |
|------------------------|-----------------------|------------------------------------|
| Evolution account      | `evolutionApi`        | sending/receiving WhatsApp messages |
| OpenRouter account      | `openRouterApi`        | all Gemini calls                   |
| Google Sheets account   | `googleSheetsOAuth2Api`| price lookup, handoff log          |
| Postgres account        | `postgres`             | chat memory                        |
| (webhook header auth)   | `httpHeaderAuth`       | authenticates incoming webhook calls |

`workflow.json` is a point-in-time export — after changing anything in the n8n UI, re-export and commit it so the repo stays in sync.

## Notes

- All customer-facing replies are in English by design.
- There is no automated test suite; changes are verified by running the workflow against test payloads and inspecting n8n's Executions list.
