---
name: hiver-manage-inbox-tags
description: >-
  Look up, search and create tags in a Hiver Shared Inbox, and apply or remove them on a
  conversation.
api: Hiver API
base_url: https://api2.hiverhq.com/v1
generated: '2026-08-22'
method: generated
source: openapi/hiver-api-openapi.json + https://developer.hiverhq.com/hiver-api
operations:
  - Inbox_inbox/list-all-the-inboxes
  - Inbox_inbox/get-tags-in-the-inbox
  - Inbox_inbox/search-tags-in-the-inbox
  - Inbox_inbox/create-tags-in-the-inbox
  - Conversations_conversations/update-conversation-in-the-inbox
---

# Manage tags in a Hiver Shared Inbox

## Steps

1. **Resolve the inbox.** `GET /inboxes` (`Inbox_inbox/list-all-the-inboxes`).
2. **Check whether the tag already exists — always do this first.**
   `GET /inboxes/{inbox_id}/tags/search?name=<name>`
   (`Inbox_inbox/search-tags-in-the-inbox`), or enumerate with
   `GET /inboxes/{inbox_id}/tags` (`Inbox_inbox/get-tags-in-the-inbox`) and follow
   `data.pagination.next_page` until it is `null`.
3. **Create only if the search missed.** `POST /inboxes/{inbox_id}/tags`
   (`Inbox_inbox/create-tags-in-the-inbox`):
   ```json
   { "name": "Sales pipeline", "color_hexcode": "#64b5f6", "description": "Sales pipeline conversations" }
   ```
   The response returns `tag_id`, `name`, `color_hexcode`, `background_hexcode`, `smid`,
   `description` — with a `200`, not a `201`.
4. **Apply or remove on a conversation.**
   `PATCH /inboxes/{inbox_id}/conversations/{conversation_id}`
   (`Conversations_conversations/update-conversation-in-the-inbox`) with
   `{"tags": {"to_apply": ["<id>"], "to_remove": ["<id>"]}}`. Returns `204`.

## Rules

- **There is no delete-tag endpoint.** Once created through the API, a tag can only be removed by
  a human in the Hiver admin panel. Step 2 is not an optimisation — it is the only protection
  against permanently littering a customer's inbox with near-duplicate tags.
- **Field names differ between read and create.** Reads give you `id` and `color_code`; the
  create response gives `tag_id` and `color_hexcode`. Normalise before comparing.
- Tag ids are numeric but returned as strings on read and as a number on create. Coerce.
- 1 RPS / 5000 per day account-wide, `429` on exhaustion, no rate-limit headers, exponential
  backoff. See `conventions/hiver-conventions.yml`.
