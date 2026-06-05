# File: distributed-email-service/email_service.py

**Date:** 2026-06-05
**Time:** 13:47

## Purpose

This file is the core implementation of a **distributed email service** — one of the system design interview reference implementations in the `sdi-implementations` repo. It provides an in-memory simulation of an email platform (think Gmail/Outlook backend) covering account management, sending/receiving, folder organization, threading, drafts, search, and read tracking.

It owns **all email lifecycle state**: creation, delivery, storage, retrieval, organization, and deletion. There is no persistence layer — everything lives in Python dicts and sets, which is intentional for an SDI teaching implementation that focuses on API design and data modeling rather than storage mechanics.

## Key Components

### `Email` (data class)

A plain value object representing an email to be sent. Fields map directly to RFC 5322 concepts: `from_addr`, `to`, `subject`, `body`, `cc`, `bcc`, `attachments`, `in_reply_to`. Mutable defaults are handled correctly with `or []` in `__init__`.

Note: `Email` is the **input** representation. Once sent, emails are stored as plain dicts (not `Email` instances) in `self.emails`.

### `EmailService` (service class)

The main service with seven internal stores:

| Store | Type | Purpose |
|-------|------|---------|
| `accounts` | `{email: name}` | User registry |
| `emails` | `{msg_id: dict}` | Canonical email store |
| `user_folders` | `{user: {folder: [msg_ids]}}` | Per-user folder organization |
| `read_status` | `{user: set(msg_ids)}` | Tracks which messages a user has read |
| `threads` | `{thread_id: [msg_ids]}` | Conversation threading |
| `msg_to_thread` | `{msg_id: thread_id}` | Reverse index for thread lookup |
| `drafts` | `{draft_id: Email}` | Draft email objects (pre-send) |

Key methods and their contracts:

- **`send(email) -> msg_id`**: The main write path. Generates a UUID, stores the email dict, assigns a thread, delivers to sender's sent folder (auto-marked read), and delivers to all recipients' inboxes. Returns the message ID.

- **`save_draft / update_draft / send_draft`**: Draft lifecycle. Drafts are stored both as `Email` objects (in `self.drafts`) and as email dicts (in `self.emails` with `is_draft: True`). `send_draft` promotes a draft to a real email by popping it from both stores and calling `send()`.

- **`list_folder(user, folder, limit, cursor) -> {emails, next_cursor, total}`**: Paginated folder listing in reverse chronological order. Uses integer-based cursor pagination.

- **`get_email(user, message_id)`**: Retrieves an email and **side-effects a mark-read**. This is the "open email" action.

- **`search(user, query, ...)`**: Linear scan with text matching on subject/body plus optional filters (from, attachments, read status). No indexing.

- **`delete(user, message_id)`**: Two-phase delete — first call moves to trash, second call from trash permanently removes.

## Patterns

**Lazy user initialization**: `_init_user()` is called at the start of most operations to ensure default folders exist. This means accounts don't strictly need `create_account()` — any operation will bootstrap the user. This is a defensive pattern common in SDI implementations where you don't want operations to fail just because setup was skipped.

**Separate input/storage representations**: `Email` objects go in, plain dicts come out. The `_store_email` method handles the conversion. This decouples the send API from the storage/query format.

**Thread assignment by reply chain**: `_assign_thread` uses `in_reply_to` to find an existing thread. If the replied-to message has a thread, the new message joins it. Otherwise, a new thread is created with `thread_id = msg_id` (the first message's ID becomes the thread ID). This is a simplified version of how Gmail threading works.

**Cursor-based pagination**: `list_folder` uses integer offsets serialized as strings. This is a teaching simplification — real systems use opaque cursors to avoid issues with concurrent inserts shifting offsets.

## Dependencies

Minimal — only stdlib:
- `uuid` for message/draft ID generation
- `datetime` for UTC timestamps

Imported by two test files (`test_email.py`, `test_email_service.py`), meaning this module is tested from two angles, likely unit vs. integration or different test authors.

## Flow

**Send path**: `send()` → `_store_email()` (Email → dict) → `_assign_thread()` (thread management) → `_add_to_folder()` for sender's "sent" + each recipient's "inbox". BCC recipients get the message in their inbox but BCC addresses are stored in the email dict — a real system would strip them from the stored record for non-sender views.

**Draft path**: `save_draft()` stores both the `Email` object and a dict representation → `update_draft()` mutates both in place → `send_draft()` pops from draft stores, removes from drafts folder, calls `send()` for normal delivery.

**Read path**: `list_folder()` returns paginated email dicts. `get_email()` returns a single email and marks it read. `get_thread()` returns all emails in a thread.

**Delete path**: `delete()` checks if message is already in trash. If yes, removes permanently. If no, calls `move_to_folder(user, message_id, "trash")`.

## Invariants

1. **Every sent message gets a UUID and UTC timestamp** — `send()` always generates both before any storage.
2. **Sender's copy is auto-marked read** — in `send()`, the sender's message ID is added to their `read_status` immediately.
3. **Thread ID equals the first message's ID** — when no `in_reply_to` exists or the replied-to message has no thread, `thread_id = msg_id`.
4. **`move_to_folder` removes from exactly one source folder** — it iterates all folders and `break`s on the first match, so a message can only exist in one folder per user.
5. **Two-phase delete** — first delete moves to trash, second delete from trash permanently removes. There's no "empty trash" bulk operation.
6. **`get_email` always marks read** — there's no way to retrieve an email without the read side-effect.

## Error Handling

Minimal and deliberate:

- `update_draft` and `send_draft` raise `ValueError("Draft not found")` if the draft ID doesn't exist.
- `get_email` returns `None` for unknown message IDs rather than raising.
- Most other methods silently no-op on missing data (e.g., `_add_to_folder` lazily creates folders, `search` skips missing message IDs with `if not e: continue`).
- No validation on email addresses, no checks that sender has an account, no permission checks that a user can access a given message. This is typical for SDI implementations where the focus is on the data model and API shape, not input validation.

## Topics to Explore

- [file] `distributed-email-service/test_email_service.py` — See how the service API is exercised end-to-end, especially threading and draft workflows
- [function] `distributed-email-service/email_service.py:search` — Linear scan search is the biggest scalability gap; worth exploring how a real system would use inverted indexes
- [general] `bcc-privacy-leak` — The current implementation stores BCC addresses in the email dict visible to all recipients, which violates BCC semantics in a real system
- [file] `distributed-email-service/plan.md` — Design decisions and trade-offs considered before implementation
- [general] `cursor-pagination-consistency` — Integer-offset cursors break under concurrent writes; worth comparing to token-based cursors used by Gmail API

## Beliefs

- `email-service-single-folder-per-user` — A message can only exist in one folder per user; `move_to_folder` removes from the source before adding to the target
- `email-service-thread-id-is-first-msg` — Thread IDs are the message ID of the thread's first message, not a separately generated identifier
- `email-service-get-email-marks-read` — `get_email` always side-effects a `mark_read`; there is no read-without-marking retrieval path
- `email-service-two-phase-delete` — First `delete()` call moves to trash; second call on a trashed message permanently removes it
- `email-service-bcc-stored-in-record` — BCC recipients are stored in the email dict alongside to/cc, which would leak BCC information to other recipients in a real system

