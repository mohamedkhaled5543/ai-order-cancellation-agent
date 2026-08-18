# AI Order Cancellation Agent

An AI-powered customer support agent built in n8n that handles order cancellation requests end-to-end via Telegram — checking business rules before cancelling, and never letting the LLM bypass them.

## What it does

A customer messages the Telegram bot to cancel an order. The AI Agent:
1. Identifies the order ID from the conversation
2. Looks up the order and checks eligibility (status must be `Processing`, payment must not be `Failed`)
3. Asks the customer to explicitly confirm before cancelling
4. Executes the cancellation and updates the order record
5. Replies with a clear summary

The agent also handles general order/customer lookups and basic calculations, with persistent conversation memory per chat.

## Architecture

```
Telegram Trigger
      │
      ▼
   AI Agent (OpenRouter LLM)
      │
      ├── Postgres Chat Memory        (conversation history per chat)
      ├── Customer Data tool          (Google Sheets)
      ├── Orders Data tool            (Google Sheets)
      ├── Calculator tool
      ├── Cancellation Checking       (sub-workflow: eligibility check)
      └── Execute Order Cancellation  (sub-workflow: performs the cancel)
      │
      ▼
Send Telegram reply
```

**Why two separate sub-workflows for cancellation?**
The LLM never decides eligibility itself. "Cancellation Checking" is the single source of truth for business rules (order status + payment status), and "Execute Order Cancellation" only runs after explicit customer confirmation — re-verifying eligibility before writing to the sheet.

## Workflows

| File | Purpose |
|---|---|
| `workflows/ai-customer-operations-assistant.json` | Main agent — Telegram trigger, LLM, memory, and all tools |
| `workflows/checking-order-status.json` | Sub-workflow — checks if an order is eligible for cancellation |
| `workflows/execute-order-cancellation.json` | Sub-workflow — re-checks eligibility, then cancels the order in the sheet |

## Setup

1. Import all 3 JSON files into n8n
2. Create credentials for:
   - **Telegram** (bot token)
   - **OpenRouter** (or swap for any LLM provider n8n supports)
   - **Postgres** (chat memory storage — e.g. free Supabase project)
   - **Google Sheets** (OAuth2)
3. In the main workflow, point the two `Execute Workflow` tool nodes to your imported sub-workflow IDs
4. Point the Google Sheets nodes at your own Orders/Customers sheets. Expected Orders columns:
   `order_id, customer_id, order_date, product_name, product_category, quantity, unit_price, total_amount, order_status, payment_status, shipping_city, estimated_delivery_date`
5. Activate all 3 workflows

## Tech stack

- **n8n** — workflow orchestration
- **OpenRouter** — LLM provider (model configurable per node)
- **Postgres (Supabase)** — chat memory
- **Google Sheets** — order/customer data store
- **Telegram Bot API** — user interface

## Notes

- Credentials are stripped from these JSON exports — reconnect your own after importing.
- Cancellation logic is duplicated across the two sub-workflows intentionally: checking is read-only and cheap to call repeatedly; execution re-verifies before any write, so a stale check can never cancel an order that changed state in between.
