---
name: hiver-post-internal-note
description: >-
  Add an internal note to a Hiver conversation, optionally @mentioning a teammate or replying to
  an existing note, without exposing anything to the customer.
api: Hiver API
base_url: https://api2.hiverhq.com/v1
generated: '2026-08-22'
method: generated
source: openapi/hiver-api-openapi.json + https://developer.hiverhq.com/hiver-api
operations:
  - Inbox_inbox/list-all-the-inboxes
  - Inbox_inbox/search-users-in-the-inbox
  - Conversations_conversations/get-a-conversation-in-the-inbox
  - Conversations_conversations/create-note-on-conversation
---

# Post an internal note on a Hiver conversation

Notes are **internal only** — they are never sent to the customer. Mentions notify the teammate.

## Steps

1. **Resolve the inbox.** `GET /inboxes` (`Inbox_inbox/list-all-the-inboxes`), keep `id`.
2. **Resolve the person you intend to mention.**
   `GET /inboxes/{inbox_id}/users/search?email=<address>`
   (`Inbox_inbox/search-users-in-the-inbox`). Only mention an address this returns — a mention of
   an address that is not a member of the inbox will not reach anybody.
3. **Confirm the conversation exists.**
   `GET /inboxes/{inbox_id}/conversations/{conversation_id}`
   (`Conversations_conversations/get-a-conversation-in-the-inbox`). To thread a reply, take the
   `parent_note_id` from the note you are replying to.
4. **Post the note.** `POST /inboxes/{inboxId}/conversations/{conversationId}/notes`
   (`Conversations_conversations/create-note-on-conversation`). Success is **`201`** and the body
   echoes the created note: `id`, `conversation_id`, `content`, `author`, `mentions`,
   `parent_note_id`, `attachments`, `created_at` (ISO 8601).

   Note the path parameter casing — this operation uses `{inboxId}` / `{conversationId}` in
   camelCase while every other operation uses `{inbox_id}` / `{conversation_id}`. Build the URL
   from the template, not from a shared helper that assumes snake_case.

## Rules

- **This is a one-way door.** Hiver publishes no endpoint to edit or delete a note, and no list
  endpoint to read notes back. A note posted in error stays, and a mention that fires cannot be
  unfired. Confirm the content and the mention list **before** the POST.
- The request body schema is **not declared in Hiver's OpenAPI**. Compose the note from the
  fields visible in the documented response example (`content`, `mentions`, `parent_note_id`) and
  verify against the response you get back.
- 1 RPS / 5000 requests per day, account-wide. No rate-limit headers. Back off exponentially on
  `429`.
- No `Idempotency-Key` exists. A timed-out POST may or may not have created the note, and you
  cannot list notes to find out — prefer a single attempt and surface the ambiguity to a human.
