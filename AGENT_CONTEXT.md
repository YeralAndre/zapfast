# ZapFast Fork - Development Context

## Goal
Improve ZapFast while keeping fixes upstream-friendly.

## Environment
- CachyOS
- KDE Plasma / Wayland
- Rust 1.98.1
- Fork: YeralAndre/zapfast
- Upstream: crmne/zapfast

## Build
Current build command on this machine:

```bash
env CC=gcc CXX=g++ CARGO_TARGET_X86_64_UNKNOWN_LINUX_GNU_LINKER=gcc cargo check
```

For test builds on this laptop (8 GB RAM):

```bash
env CC=gcc CXX=g++ CARGO_TARGET_X86_64_UNKNOWN_LINUX_GNU_LINKER=gcc CARGO_BUILD_JOBS=1 cargo test --lib -- <test_name>
```

## Current branch
`fix/empty-chat-rows`

## Confirmed root issue

`Worker::emit_chats` currently contains:

```rust
chats.retain(|chat| chat.last.is_some() || self.canonical_str(&chat.id) == chat.id);
```

For canonical direct JIDs such as `<phone>@s.whatsapp.net`, the canonical comparison (`self.canonical_str(&chat.id) == chat.id`) evaluates to `true`. Consequently, empty direct-chat rows survive indefinitely in `self.chats` even when there is no message, no activity, and no user state.

The original logic was introduced in the commit with `early_privacy_id_mute_reaches_the_canonical_chat_without_a_duplicate` to remove mapped `@lid` privacy-ID duplicates, not to intentionally expose every canonical empty direct chat from the phone's address book or group participant lists.

## Important regression discovered

Filtering empty direct chats only in `Worker::emit_chats` is not sufficient.

`Action::StartChat` creates an intentionally empty direct chat in `App::chats` and sets `App::open_chat`.

If a later `Event::Chats` omits it (because the worker sees `last == None` and `last_activity == 0` in SQLite):
- `self.chats` is replaced with `chats`.
- The open chat disappears from `self.chats`.
- `self.open_chat` becomes `None` (view closes unexpectedly).

Therefore, the eventual implementation must preserve explicitly started/open chats across updates.

## Test state

Six regression tests are present in the test suite:

1. `canonical_direct_chat_without_messages_or_activity_is_not_emitted_in_chats`: **FAIL** (reproduces the bug on current `main`).
2. `early_privacy_id_mute_reaches_the_canonical_chat_without_a_duplicate`: **PASS** (existing privacy-ID deduplication test passes unchanged).
3. `valid_empty_group_remains_visible_in_chats`: **PASS** (empty group is retained).
4. `valid_empty_channel_remains_visible_in_chats`: **PASS** (empty channel is retained).
5. `direct_chat_with_activity_but_no_local_messages_remains_visible`: **PASS** (chat with `last_activity > 0` is retained for on-demand fetch).
6. `direct_chat_created_via_start_chat_does_not_disappear_on_empty_chats_event`: **FAIL** (proves that omitting empty chats in `Event::Chats` closes `open_chat` in `App`).

## Production code status

No production fix has been approved or implemented yet. Source code changes are strictly test-only.

## Proposed solution status

Antigravity proposed a collaborative Worker/App approach (Option D), but it is NOT approved yet. It must be reviewed before implementation.

## Next step

Review the implementation boundary and design the smallest safe fix before touching production code.
