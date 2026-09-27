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

env CC=gcc CXX=g++ CARGO_TARGET_X86_64_UNKNOWN_LINUX_GNU_LINKER=gcc cargo check

Build currently succeeds.

## Confirmed issues to investigate

### 1. Chat classification
- Archived chats, channels and communities are not separated correctly.
- Contacts related to groups may appear as standalone chats.
- Sidebar filters exist but classification appears incomplete.

### 2. Archive sync
- Archived chats sometimes remain in archived state until manually unarchived.
- Remote state reconciliation needs investigation.

### 3. Stickers
- Some sticker thumbnails remain placeholders.
- Sticker sending can be slow.
- Recent/favorites behavior is incomplete or missing.

### 4. Image viewer
- Mouse wheel zoom works.
- Touchpad pinch zoom does not.

### 5. KDE Secret Service compatibility
Previously reproduced on ZapFast 0.14.0:
- ZapFast created its keyring item but stored an empty secret under ksecretd.
- Manual valid 32-character secret allowed ZapFast to continue normally.
- ksecretd also crashed once with SIGSEGV in QCA/OpenSSL.
- Current source version is 0.17.0; this should be re-tested before touching code.

## Development rules
- One bug per branch.
- Small upstream-friendly fixes.
- Avoid unrelated refactors.
- Preserve project style.
- Add tests when practical.
- Run cargo fmt, cargo clippy and cargo test before PRs.

## Next task
Repository reconnaissance only.
Do not modify code until architecture has been mapped.
