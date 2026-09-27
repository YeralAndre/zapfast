# ZapFast Development Handoff

## Date
2026-09-27

## Branch
`fix/empty-chat-rows`

## Goal
Remove phantom empty direct-chat rows without breaking legitimate empty conversations or `StartChat`.

## What was investigated

During this investigation, the following components and data flows were analyzed:
- `Worker::emit_chats`: Evaluates `chats` from SQLite archive and retains or filters items before emitting `Event::Chats`.
- `Worker::apply_history`: Ingests `wa::HistorySync` chunks, creates rows in the `chats` table via `Archive::upsert_chat`, and emits `Event::ChatUpdated` for entries in `filed`.
- `Worker::canonical` and `canonical_str`: Normalizes JIDs and resolves known `@lid` privacy IDs to standard phone-number JIDs (`@s.whatsapp.net`).
- `@lid` mappings: Evaluates how privacy IDs are learned and mapped to canonical phone numbers.
- `HistorySync`: Examined how WhatsApp sends `wa::Conversation` protobuf objects with zero messages and zero activity for address-book contacts and group participants.
- `App::matching_contacts`: Examined contact search logic in `src/app.rs`, which explicitly excludes any contact whose ID already exists in `self.chats`.
- `StartChat` and `EnsureChat`: Traced `Action::StartChat`, `Command::EnsureChat`, and how an empty chat is initiated when a user selects a contact from the UI.
- `Event::Chats` handling: Analyzed how `App::handle_events` replaces `self.chats = chats` and closes `self.open_chat` if the open chat ID is absent from the incoming list.

## Confirmed behavior

In `src/backend/worker.rs`, `Worker::emit_chats` currently executes:

```rust
chats.retain(|chat| chat.last.is_some() || self.canonical_str(&chat.id) == chat.id);
```

For any standard direct contact JID (`<phone>@s.whatsapp.net`), `self.canonical_str(&chat.id)` returns the exact same string, so `self.canonical_str(&chat.id) == chat.id` evaluates to `true`.

Consequently:
- The expression `chat.last.is_some() || true` always evaluates to `true`.
- The `chat.last.is_some()` check is completely bypassed for all canonical direct contacts.
- Address-book contacts and group participants ingested with `last == None` and `last_activity == 0` remain indefinitely in `self.chats` and appear as standalone rows in the main chat list.
- Git history confirms that this condition was originally added to purge obsolete `@lid` alias rows after their canonical mapping was established, not to expose empty contacts as active chats.

## Legitimate empty-chat cases discovered

Not every chat with `last == None` is an unwanted ghost row. Legitimate cases that must remain visible include:
- **Empty groups**: `chat.kind == ChatKind::Group` (`@g.us`). Memberships exist before messages are sent or downloaded.
- **Empty channels and newsletters**: `chat.is_channel()` (`@newsletter`). Followed feeds must appear in the Channels tab.
- **Explicit user or sync states**:
  - Pinned chats: `chat.pinned || chat.pinned_at > 0`
  - Favorite chats: `chat.favorite`
  - Archived chats: `chat.archived`
  - Locked chats: `chat.locked`
  - Unread chats: `chat.looks_unread()` (`chat.unread > 0 || chat.marked_unread`)
- **Remote phone history (`last_activity > 0`)**: Legitimate chats where WhatsApp provided an activity timestamp but no initial local messages. These must be listed so ZapFast can trigger on-demand history fetch (`fetch_older`) upon opening.
- **Manually started direct chats (`StartChat`)**: A contact opened by the user to compose a new message, where `last == None` until the first message is dispatched.

## Tests added

Six regression tests were added and validated:

1. `canonical_direct_chat_without_messages_or_activity_is_not_emitted_in_chats` in `src/backend/worker.rs`:
   - Purpose: Validates that an address-book contact with `last == None` and `last_activity == 0` is omitted from `Event::Chats`.
   - Result: **FAIL** on current `main` (panicked at assertion, reproducing the reported bug).
2. `early_privacy_id_mute_reaches_the_canonical_chat_without_a_duplicate` in `src/backend/worker.rs`:
   - Purpose: Verifies that existing `@lid` deduplication behavior remains intact.
   - Result: **PASS** on current `main`.
3. `valid_empty_group_remains_visible_in_chats` in `src/backend/worker.rs`:
   - Purpose: Verifies that groups without messages are retained.
   - Result: **PASS** on current `main`.
4. `valid_empty_channel_remains_visible_in_chats` in `src/backend/worker.rs`:
   - Purpose: Verifies that channels without messages are retained.
   - Result: **PASS** on current `main`.
5. `direct_chat_with_activity_but_no_local_messages_remains_visible` in `src/backend/worker.rs`:
   - Purpose: Verifies that direct chats with `last_activity > 0` but no local messages are retained.
   - Result: **PASS** on current `main`.
6. `direct_chat_created_via_start_chat_does_not_disappear_on_empty_chats_event` in `src/app.rs`:
   - Purpose: Verifies that an empty chat opened via `Action::StartChat` is not dropped and closed when `Event::Chats` arrives without it.
   - Result: **FAIL** on current `main` (panicked because `open_chat` became `None`).

## Critical architectural discovery

Filtering empty direct chats exclusively in `Worker::emit_chats` is insufficient and creates a critical UX regression:
- The backend worker runs on an independent Tokio runtime thread and does not observe UI-local state such as `App::open_chat` or unsaved composer text.
- If `Worker::emit_chats` filters out all canonical chats with `last == None` and `last_activity == 0`, a newly started chat (`Action::StartChat`) will be excluded from the next `Event::Chats`.
- When `App::handle_events` receives `Event::Chats(chats)`, it executes `self.chats = chats`.
- Because the newly opened chat is absent from `chats`, `self.chat(&open)` returns `None`, and `self.open_chat` is immediately set to `None`, closing the conversation while the user is actively typing.

## Candidate implementation approaches

### Approach A: Worker-only filtering
Filter empty canonical direct chats solely inside `Worker::emit_chats`.
- Status: **Rejected / Insufficient**.
- Reason: Breaks `StartChat`, closing newly opened chats upon background events.

### Approach B: App::visible_chats-only filtering
Allow the worker to emit all chats to `App::chats`, but hide empty contacts in `App::visible_chats`.
- Status: **Rejected / Insufficient**.
- Reason: Empty contacts remain in `self.chats`. `App::matching_contacts` explicitly filters `!self.chats.iter().any(|chat| chat.id == contact.id)`, which prevents those contacts from appearing in contact search results when attempting to start a new chat.

### Approach C: Broad persisted-vs-visible model separation
Refactor database schemas, models, and worker events to formally decouple contacts from conversation threads.
- Status: **Not Recommended**.
- Reason: Too broad and invasive for a focused, safe bugfix; violates the rule against unnecessary general refactoring.

### Approach D: Collaborative Worker + App solution
- Worker adjustment: In `Worker::emit_chats`, filter out canonical direct chats that lack messages, activity, explicit flags (pin, favorite, archive, lock, unread), or drafts. Also avoid emitting `ChatUpdated` for inactive contacts in `apply_history`.
- App adjustment: In `App::handle_events` for `Event::Chats`, preserve `self.open_chat` in `self.chats` if it is currently open in the interface but not yet present in the received list.
- Status: **Current candidate, NOT approved yet**. Requires review before implementation.

## Exact next task

The next session must:
1. Read `AGENT_CONTEXT.md`.
2. Read `docs/SESSION_HANDOFF.md`.
3. Inspect the newly added tests in `src/backend/worker.rs` and `src/app.rs`.
4. Review the candidate implementation boundary (Option D vs alternatives).
5. Do not modify production code until the proposed minimal fix has been formally approved.

## Build notes for this laptop

Current required linker workaround on this machine:

```bash
env CC=gcc CXX=g++ CARGO_TARGET_X86_64_UNKNOWN_LINUX_GNU_LINKER=gcc cargo ...
```

The laptop has 8 GB RAM. Full parallel builds cause severe memory pressure and swap thrashing.
For this laptop, always prefer:

```bash
CARGO_BUILD_JOBS=1
```

Future development will continue on a more powerful machine.
