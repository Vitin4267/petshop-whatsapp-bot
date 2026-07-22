# PetShop WhatsApp AI Assistant

A WhatsApp customer-service bot for a pet shop, built as a single [n8n](https://n8n.io) workflow. It handles greetings, price questions, scheduling, complaints, and image/audio messages — and is deliberately architected to keep LLM costs down, degrade gracefully under load, and never leave a customer without a reply.

## How it works

```
Webhook (Evolution API)
→ Rate Limiter: 10 req/10s per contact, truncates overly long messages
   → over the limit → polite "please wait" reply (never silent)
→ Message Debounce: buffers rapid-fire fragments per contact (Postgres-backed),
   waits for a few seconds of silence, then joins them into one input
→ Media detection → image/audio described by Gemini (with a disguised-failure guard:
   a 200 response with no real content falls through to a friendly reply, not a crash),
   other file types get a static reply
→ Intent Classifier (Gemini 2.5 Flash Lite, temperature 0, tool-based conversation
   history on demand): greeting | out_of_scope | exception
→ greeting / out_of_scope → instant canned reply, no further LLM call
→ exception → full AI Agent (Postgres memory + Google Sheets tools) handles pricing,
   scheduling, complaints, and anything that needs real reasoning or a spreadsheet lookup
```

## Why it's built this way

- **Cost-optimized by design.** Most incoming messages (greeting, out-of-scope) never reach the expensive conversational agent — a cheap, fast classifier triages them first, and only genuinely ambiguous or pricing/scheduling cases escalate to the full agent with memory and tools.
- **One thought, one reply.** WhatsApp users routinely split a single request across several messages sent seconds apart. A Postgres-backed debounce buffers fragments per contact and only processes them once the burst goes quiet, instead of firing a separate (wrong) reply per fragment.
- **Classifier reasons from real conversation, not its own past decisions.** Early on, the classifier had its own conversation memory — which meant it saw its *own prior categorizations* as context and started parroting them regardless of the current message. Fixed by removing that memory and giving it, instead, an on-demand tool that reads the *real* customer/agent conversation history from Postgres, filtered by a session ID that's fixed in the tool's config (not left to the model) — so it can never be tricked into pulling another customer's conversation.
- **Deterministic where it has to be.** The classifier runs at `temperature: 0`. It's a routing decision, not creative writing — the same message should always land in the same category.
- **Never refuses.** The classifier's structured-output step failed once when the model replied with a plain-text apology instead of a category. The system prompt now explicitly forbids refusals: it must always emit one of the three categories, defaulting to `exception` when unsure.
- **Fails loud, not silent.** Every LLM/HTTP call (classifier, media analysis, agent, outbound message) has retry-on-failure configured, and the media-analysis step specifically checks for a disguised failure (HTTP 200 with an error payload instead of real content) rather than trusting a 200 status blindly. Hitting the rate limit gets you a friendly reply, not dead air.
- **Security-conscious.** Per-contact rate limiting, input length caps, an explicit prompt-injection defense in the agent's system prompt (customer text is treated as data, never as instructions), and every Postgres tool query has its contact filter hardcoded in the node config rather than left for the model to supply.
- **Session-aware without a database.** Lightweight per-contact state (already greeted? already asked prices?) lives in n8n's own persisted workflow storage — no Redis, no extra infra — and expires after 24h so it never goes stale.
- **Group-safe.** The connected WhatsApp instance ignores group chats, so multiple people in the same group can't bleed into one shared conversation context.

## Stack

- **n8n** — workflow engine (self-hosted, Docker)
- **Evolution API** — WhatsApp connector
- **OpenRouter** (Gemini 2.5 Flash Lite) — intent classification, media understanding, and the fallback conversational agent
- **Google Sheets** — price list lookup + human-handoff log
- **Postgres** — chat memory for the fallback agent, message debounce buffer, and the classifier's on-demand conversation-history tool

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

The message debounce buffer needs one extra table in the Postgres database used for `Postgres account`:

```sql
CREATE TABLE IF NOT EXISTS msg_buffer (
  id BIGSERIAL PRIMARY KEY,
  contact TEXT NOT NULL,
  part TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX IF NOT EXISTS idx_msg_buffer_contact ON msg_buffer(contact, created_at);
```

## Notes

- All customer-facing replies are in English by design.
- There is no automated test suite; changes are verified by running the workflow against test payloads and inspecting n8n's Executions list.
- The message debounce buffer isn't fully concurrency-safe under heavy simultaneous load from the same contact — fine at demo/portfolio scale; a production deployment would want an explicit row lock (`SELECT ... FOR UPDATE`) around the flush check.
