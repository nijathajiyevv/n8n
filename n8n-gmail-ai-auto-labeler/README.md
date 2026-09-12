# Gmail AI Auto-Labeler (n8n)

An n8n workflow that reads incoming Gmail messages, classifies them with an LLM, and applies the matching Gmail label automatically.

## The problem

Inbox volume outpaces manual triage. Newsletters, receipts, security alerts, shipping updates, and work mail all land in one stream. Manual labeling does not scale and gets abandoned within days. Gmail's native filters only match static rules (sender, keyword), which fail on messages with inconsistent formatting or new senders. The result is an inbox that is either unsorted or requires continuous manual maintenance.

Labeling matters because it is the precondition for everything downstream: search, archiving rules, follow-up automation, and reporting. An unlabeled inbox cannot be filtered, batched, or delegated. This workflow removes the manual step by having a model read each email and assign one of a fixed set of categories, then routing the message to the correct Gmail label with zero human input after setup.

## What this does

1. Polls Gmail every minute for unread messages.
2. Extracts subject, sender, and a body preview from each message.
3. Sends that data to an AI Agent node backed by an OpenRouter-hosted model.
4. The model returns exactly one category name from a fixed list, nothing else.
5. A parsing step validates the output against the allowed category list and defaults to `Other` if the response is empty or unrecognized.
6. A Switch node routes the message to one of eleven paths.
7. A dedicated Gmail node applies the corresponding label on each path.

## Architecture

```
Gmail Trigger (poll unread, every minute)
        │
        ▼
Prepare Email Data (Code node — normalizes subject/from/body into a compact object)
        │
        ▼
AI Agent  ←── OpenRouter Chat Model (anthropic/claude-3-haiku, temp 0, max 100 tokens)
        │
        ▼
Parse Category (Code node — validates model output against allowed list, falls back to "Other")
        │
        ▼
Route by Category (Switch node — 11 branches)
        │
        ├─ Label: Newsletters
        ├─ Label: Advertisings
        ├─ Label: Security
        ├─ Label: Orders_Shipping
        ├─ Label: Payments_Receipts
        ├─ Label: Appointments
        ├─ Label: Work
        ├─ Label: Events_Meetups
        ├─ Label: Real_Estate
        ├─ Label: Social_Notifications
        └─ Label: Other
```

## Categories

| Category | Description |
|---|---|
| Newsletters | Recurring digest or content emails from subscribed publications and paid newsletter platforms |
| Advertisings | Marketing and upsell emails from vendors or services in use |
| Security | Sign-in alerts, device alerts, breach warnings, card/account expiry notices |
| Orders_Shipping | Order confirmations, shipping updates, returns, delivery complaints |
| Payments_Receipts | Payment receipts, invoices, direct debit notices, subscription billing |
| Appointments | Booking confirmations for services (medical, fitness, personal care, etc.) |
| Work | SaaS notifications, webhooks, tool alerts, workplace platform notifications |
| Events_Meetups | Course sessions, demos, meetups, conferences, webinars |
| Real_Estate | Property listings or real estate platform notifications |
| Social_Notifications | Social platform, group, or community digest notifications |
| Other | Fallback for anything unmatched or on parse failure |

The published prompt uses generic category descriptions, not real sender/vendor names. The original build was tuned with actual senders from one inbox — that list was stripped before publishing since it doubles as a fingerprint of the account holder's services, habits, and location. Add your own sender examples locally; do not commit them to a public repo.

## Build steps

**1. Connect OpenRouter**
Added OpenRouter as the LLM provider via the `lmChatOpenRouter` credential type. Model set to `anthropic/claude-3-haiku`, temperature `0` (deterministic classification, not creative generation), max tokens `100` (the output is a single category label, not prose).

**2. Connect Gmail**
Two separate Gmail integration points:
- `gmailTrigger` node polling unread mail every minute.
- Eleven `gmail` nodes (one per category) that apply a label to the message by `messageId`/`threadId`.

**3. Build the AI Agent**
The agent node holds a single fixed prompt: it lists all ten categories with concrete examples per category, then instructs the model to respond with the category name only, no punctuation, no explanation, no markdown. This constraint exists because the raw output is parsed downstream by a strict string-matching function, not by free-form NLP — the prompt has to force the model into a machine-parseable format.

**4. Parse and validate**
The `Parse Category` code node strips punctuation from the model's raw output, checks it against the allowed list (exact match, then substring match as fallback), and if nothing matches, sets the category to `Other` and records a debug reason (`EMPTY_OUTPUT`, `NO_MATCHING_CATEGORY_IN_RESPONSE`, or `CATEGORY_NOT_IN_ALLOWED_LIST`). This step exists because LLM output is not guaranteed to conform to instructions on every call — the validation layer is what keeps a malformed response from breaking the routing step or silently mislabeling mail.

**5. Route and label**
A Switch node with strict, case-sensitive equality checks on the `category` field sends each item down exactly one of eleven branches, each terminating in a Gmail node that applies the corresponding label.

## Setup

1. Import `Gmail_AI_Auto-Labeler.json` into n8n.
2. Create the Gmail OAuth2 credential in your own n8n instance and attach it to the Gmail Trigger node and all eleven label nodes. The credential fields in the exported JSON are placeholders (`YOUR_GMAIL_CREDENTIAL_ID`) — n8n will prompt you to select or create a real credential on import.
3. Create the corresponding labels in Gmail first: Newsletters, Advertisings, Security, Orders_Shipping, Payments_Receipts, Appointments, Work, Events_Meetups, Real_Estate, Social_Notifications, Other.
4. In each of the eleven Gmail label nodes, replace the placeholder in `labelIds` (e.g. `REPLACE_WITH_YOUR_NEWSLETTERS_LABEL_ID`) with the actual label ID from your Gmail account. Label IDs are account-specific and cannot be reused across accounts — find them via the Gmail node's built-in label picker in the n8n editor, or via the Gmail API `labels.list` endpoint.
5. Create an OpenRouter API credential and attach it to the OpenRouter Chat Model node.
6. Edit the category examples inside the AI Agent node's prompt to match your own senders and use cases if you want higher classification accuracy. Keep any real sender/vendor names out of the version you publish.
7. Activate the workflow.

## Before publishing your own fork

The original export contained three categories of account-specific data, now stripped from this version: Gmail/OpenRouter credential IDs, Gmail label IDs, and a hardcoded list of real newsletter/vendor/service names in the AI Agent prompt (which functioned as an identifying fingerprint of the account owner). If you customize this workflow with your own senders and export it again, repeat this check before pushing.

## Known limitations

- Category examples are hardcoded from one inbox. New senders outside the given examples rely on the model generalizing from the category description, not from a memorized sender list.
- The Switch node uses case-sensitive strict equality — if the model prompt is edited and a category name is renamed without updating the Switch conditions, that branch silently stops matching and everything falls through to `Other`.
- No retry logic on the AI Agent call. A failed or empty model response goes straight to `Other` via the fallback in `Parse Category`, rather than being retried.
- Runs on unread mail only. Mail marked read before the workflow processes it will not be labeled.
