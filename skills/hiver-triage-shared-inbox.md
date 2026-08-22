---
name: hiver-triage-shared-inbox
description: >-
  Find open conversations in a Hiver Shared Inbox, assign each to the right agent and apply
  routing tags, using only the operations Hiver publishes in its v1 REST contract.
api: Hiver API
base_url: https://api2.hiverhq.com/v1
generated: '2026-08-22'
method: generated
source: openapi/hiver-api-openapi.json + https://developer.hiverhq.com/hiver-api
operations:
  - Inbox_inbox/list-all-the-inboxes
  - Inbox_inbox/get-all-users-in-the-inbox
  - Inbox_inbox/get-tags-in-the-inbox
  - Conversations_conversations/get-conversations-in-the-inbox
  - Conversations_conversations/get-a-conversation-in-the-inbox
  - Conversations_conversations/update-conversation-in-the-inbox
---

# Triage a Hiver Shared Inbox

Route unassigned conversations in a Shared Inbox to the right agent and tag them.

## Before you start

- Authenticate with `Authorization: Bearer <api-key>`. The key is created by a Hiver
  **administrator** under Admin Panel -> Integrations -> Developer APIs and must be toggled on.
  It carries admin privileges — treat it as a root credential.
- **You are limited to 1 request per second per account and 5000 requests per day.** There are no
  rate-limit response headers, so you cannot read your remaining budget. Pace yourself at or
  under 1 RPS rather than discovering the ceiling. On `429`, back off exponentially; continuous
  retrying while still 429ing can get the key or the client IP blacklisted.

## Steps

1. **Find the inbox.** `GET /inboxes` (`Inbox_inbox/list-all-the-inboxes`). Match on
   `data.results[].email` (for example `support@yourcompany.com`) and keep `id`.
   The response is paginated: follow `data.pagination.next_page` until it is `null`.
2. **Load the roster once.** `GET /inboxes/{inbox_id}/users`
   (`Inbox_inbox/get-all-users-in-the-inbox`) and cache it. At 1 RPS you cannot afford to
   re-read the roster per conversation. If you already know the address, use
   `GET /inboxes/{inbox_id}/users/search?email=...` instead.
3. **Load the tag vocabulary once.** `GET /inboxes/{inbox_id}/tags`
   (`Inbox_inbox/get-tags-in-the-inbox`). Note the shape drift: reads return `id` and
   `color_code`, while tag creation returns `tag_id` and `color_hexcode` for the same object.
4. **List conversations.** `GET /inboxes/{inbox_id}/conversations`
   (`Conversations_conversations/get-conversations-in-the-inbox`). Use `limit` (10–100),
   `sort_by`, `sort_order` and `next_page`. Each row carries `status`, `assignee`, `tag_ids`
   and `gmail_thread_id`.
5. **Read the current state before you change it.**
   `GET /inboxes/{inbox_id}/conversations/{conversation_id}`
   (`Conversations_conversations/get-a-conversation-in-the-inbox`). Store `status`, `assignee`
   and `tag_ids`. **This read is the only rollback material Hiver gives you** — there is no undo
   endpoint and no revision history. `conversation_id` accepts either a Hiver conversation id or
   a Gmail thread id.
6. **Apply the change.** `PATCH /inboxes/{inbox_id}/conversations/{conversation_id}`
   (`Conversations_conversations/update-conversation-in-the-inbox`) with only the blocks you are
   changing:
   ```json
   { "status": {"name": "open"},
     "assignee": {"email": "agent@yourcompany.com"},
     "tags": {"to_apply": ["<tag-id>"], "to_remove": []} }
   ```
   A success returns **`204 No Content`** — there is no body to confirm against, so re-read with
   step 5 if you need proof.

## Rules

- **No idempotency.** Hiver publishes no `Idempotency-Key` header. If a PATCH times out, re-read
  with step 5 before retrying; do not blind-retry a write.
- **Reversal is manual.** A PATCH is undone only by issuing a second PATCH carrying the values
  you captured in step 5. Hiver states no window and offers no server-side undo.
- **Errors come in two shapes**: `{"errors":[{"message":"..."}]}` and `{"Message":"..."}`.
  Handle both spellings. `401` and `403` share the same title in Hiver's own table, so treat
  either as "fix the credential or the permission", never as retryable.
- Do not invent status names. Read the values Hiver returns in step 5 and reuse them.
