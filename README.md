# PetShop WhatsApp AI Assistant

A WhatsApp customer-service bot for a pet shop, built as a single [n8n](https://n8n.io) workflow. It handles greetings, price questions, scheduling, complaints, and image/audio messages, while keeping LLM costs down by only invoking a full agent for the cases that actually need one.

## How it works

```
Webhook (Evolution API)
→ Rate Limiter: 10 req/10s per contact, truncates overly long messages
→ Media detection → image/audio described by Gemini, other file types get a static reply
→ Intent Classifier (Gemini 2.5 Flash Lite, no memory/tools): greeting | services | out_of_scope | exception
→ greeting / services / out_of_scope → instant canned reply, no further LLM call
→ exception → full AI Agent (Postgres memory + Google Sheets tools) handles scheduling,
   complaints, and anything that needs real reasoning or a spreadsheet lookup
```

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
