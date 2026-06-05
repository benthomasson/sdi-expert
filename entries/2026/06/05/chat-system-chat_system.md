# File: chat-system/chat_system.py

**Date:** 2026-06-05
**Time:** 13:21

# `chat-system/chat_system.py`

## Purpose

This file is a **single-server, in-memory chat system** implementing the core concepts you'd discuss in a "Design a Chat System" interview. It owns all messaging state — connections, conversations, message routing, presence, read receipts, and group management — in one `ChatServer` class. It's a pedagogical implementation: no network layer, no persistence, no sharding — just the domain logic that a real system like WhatsApp or Slack would distribute across many services.

## Key Components

### Data Models

**`Message`** — The central data unit. Carries both content and ordering metadata:
- `sequence_number`: per-conversation monotonic counter for total order within a conversation
- `lamport_timestamp`: global logical clock for cross-conversation causal ordering
- `edited` / `deleted`: soft-state flags — deleted messages replace content with `"[deleted]"` rather than being removed from the list

**`Conversation`** — Container for an ordered message list plus participant set. The `next_sequence` counter is the source of truth for sequence number assignment. Serves both DMs and groups (distinguished by `is_group`).

**`GroupInfo`** — Group metadata separated from the conversation itself. Tracks `members`, `admins`, `creator_id`. The membership set here is the authoritative one for permission checks; `Conversation.participants` mirrors it.

**`UserConnection`** — Per-user connection state with two queues: `inbox` (for online/away users) and `offline_queue` (buffered until reconnect). This models the fan-out delivery pattern where the server decides routing based on presence.

### `ChatServer`

The monolith. Key index structures:

| Field | Type | Purpose |
|-------|------|---------|
| `messages` | `dict[str, Message]` | Global message lookup by ID — enables edit/delete/mark-read by ID |
| `conversations` | `dict[str, Conversation]` | All conversations keyed by computed or generated ID |
| `read_cursors` | `dict[tuple, int]` | `(user_id, conv_id) → last_read_sequence` — sequence-based, not message-ID-based |
| `user_conversations` | `dict[str, set]` | Reverse index: user → their conversation IDs |
| `contacts` | `dict[str, set]` | Bidirectional contact graph, built implicitly by messaging |
| `lamport_clock` | `int` | Single global Lamport counter incremented on every message send |

### Key Methods

**`send_message`** — The DM path. Creates the conversation lazily on first message, assigns sequence + Lamport timestamps, delivers via `_deliver`, auto-marks the sender's own message as read, and builds up the contact graph and conversation index as side effects.

**`send_group_message`** — The group path. Validates sender membership, then fans out to all group members except the sender via `_deliver`.

**`_deliver`** — The routing decision point. Online/away users get messages in their `inbox`; offline users get them in `offline_queue`. This is the write-path analog of push vs. pull.

**`connect` / `disconnect`** — Manage presence transitions and flush the offline queue to inbox on reconnect. Both generate `SYSTEM` messages to the user's contacts — this is presence notification fan-out.

**`get_history`** — Cursor-based pagination using `bisect` for O(log n) cursor lookup. Supports both forward and backward traversal. Returns `(page, next_cursor)` where `next_cursor` is `None` when exhausted.

## Patterns

**Conversation ID as canonical key for DMs**: `_dm_conversation_id` sorts the two user IDs lexicographically and produces `"dm:{u1}:{u2}"`. This guarantees exactly one conversation per pair regardless of who messages first — a classic deduplication technique.

**Lamport clocks for causal ordering**: Every `send_message` and `send_group_message` increments a global `lamport_clock`. This gives a total order across all conversations on this server. In a distributed version, each server would maintain its own clock and merge on receipt.

**Sequence numbers for per-conversation ordering**: Independent from Lamport timestamps. Sequence numbers are scoped to a conversation and used for read cursors and pagination. This separation is deliberate — you want local dense ordering for pagination but global logical ordering for consistency.

**Lazy resource creation**: Conversations, connections, contacts, and conversation indices are all created on first use. There's no explicit "create DM" step — `send_message` handles it.

**Soft deletes**: `delete_message` sets `deleted=True` and overwrites content with `"[deleted]"` but keeps the message in the list. This preserves sequence number continuity and avoids holes in pagination.

**Dual-queue delivery**: The `inbox` / `offline_queue` split models the real-world pattern where a chat server buffers messages for disconnected clients and flushes them on reconnect, rather than requiring the client to poll.

## Dependencies

**Imports**: All stdlib — `uuid` for message/group IDs, `bisect` for pagination cursor lookup, `dataclasses` for data models, `enum` for status/type enums, `typing` for `Optional`.

**Imported by**: `test_chat_system.py` — the test suite. No other modules depend on this; it's a self-contained implementation.

## Flow

A typical message lifecycle:

1. **Sender calls `send_message`** → conversation created if needed → Lamport clock incremented → sequence number assigned → `Message` constructed and appended to conversation
2. **`_deliver` called for recipient** → checks recipient's `UserStatus` → routes to `inbox` (online/away) or `offline_queue` (offline)
3. **Recipient reconnects** (`connect`) → `offline_queue` flushed to `inbox` → presence notification sent to contacts
4. **Recipient reads messages** → calls `mark_read` with a message ID → server resolves to sequence number → updates `read_cursors`
5. **Unread count queried** → `get_unread_count` computes `last_seq - last_read` — O(1)

For groups, step 2 fans out to all members except sender.

## Invariants

- **Sequence numbers are dense and monotonic** within a conversation — no gaps, no reordering. Assigned by `next_sequence` counter.
- **Lamport timestamps are globally monotonic** — strictly increasing across all message sends on this server.
- **DM conversation IDs are deterministic and symmetric** — `_dm_conversation_id("alice", "bob") == _dm_conversation_id("bob", "alice")`.
- **Read cursors only advance** — `mark_read` has a `msg.sequence_number > current` guard preventing regression.
- **Group membership is checked on send** — `send_group_message` raises `ValueError` if the sender isn't a member.
- **Group size capped at 500** — `add_member` enforces this limit.
- **Typing indicators auto-expire after 5 seconds** — `get_typing_users` prunes stale entries.
- **Contacts are always bidirectional** — `send_message` adds both directions.

## Error Handling

Minimal and deliberate:

- **`send_group_message`**: raises `ValueError` if sender is not a group member
- **`add_member`**: raises `ValueError` if group has 500+ members
- **`edit_message` / `delete_message`**: raise `PermissionError` if `user_id != sender_id`
- **Missing keys** in `self.messages`, `self.groups`, `self.conversations`: will raise `KeyError` — no defensive checks. The caller is expected to pass valid IDs.
- **No validation** on content, user IDs, or timestamps. This is an implementation of the routing/ordering/delivery layer, not the API validation layer.

## Topics to Explore

- [file] `chat-system/test_chat_system.py` — See what scenarios the tests cover: edge cases in pagination, offline delivery, group membership transitions
- [file] `chat-system/plan.md` — Understand the design rationale and what tradeoffs were considered before implementation
- [function] `chat-system/chat_system.py:get_history` — The cursor-based pagination logic has subtle edge cases around forward vs backward traversal and cursor position resolution
- [general] `message-ordering-distributed` — How this single-server Lamport clock approach would change with multiple servers (vector clocks, hybrid logical clocks, or server-assigned timestamps with tie-breaking)
- [general] `fan-out-on-write-vs-read` — This implementation does fan-out-on-write (pushing to each member's queue); explore when fan-out-on-read (pull model) is preferable, especially for large groups

## Beliefs

- `chat-dm-conversation-dedup` — DM conversations use a deterministic sorted-pair ID (`dm:{min}:{max}`) guaranteeing exactly one conversation per user pair
- `chat-dual-ordering-scheme` — Messages carry both per-conversation sequence numbers (for pagination/read-cursors) and global Lamport timestamps (for causal ordering), serving different purposes
- `chat-offline-queue-flush-on-connect` — When a user connects, their `offline_queue` is drained into `inbox` in FIFO order, preserving message arrival ordering
- `chat-read-cursors-monotonic` — `mark_read` only advances the read cursor; it silently ignores attempts to set a lower sequence number
- `chat-soft-delete-preserves-sequence` — Deleted messages remain in the conversation list with `deleted=True` and content replaced, preserving sequence number continuity for pagination

