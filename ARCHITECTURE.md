# ZapFast Architecture & Operational Model

This document outlines the high-level architecture of ZapFast and its operational interaction with the protocol layer (`whatsapp-rust`).

## System Architecture Overview

ZapFast is structured into three primary layers:

```
┌─────────────────────────────────────────────────────────┐
│                       UI Layer                          │
│          (egui / eframe / fastframe / src/ui/*)         │
│   - Immediate-mode rendering                            │
│   - Emits Action / Command to Backend                   │
│   - Receives Event via Waker                            │
└──────────────────────────┬──────────────────────────────┘
                           │ (Command / Event channels)
┌──────────────────────────▼──────────────────────────────┐
│                     Backend Layer                       │
│              (Tokio Runtime / src/backend/*)            │
│   - Worker (src/backend/worker.rs)                      │
│   - Owns whatsapp-rust Client/Bot session               │
│   - Coordinates AppState sync (RegularLow, High, etc.)  │
│   - Ingests & translates incoming protobuf messages     │
└───────────────┬─────────────────────────┬───────────────┘
                │                         │
┌───────────────▼──────────┐   ┌──────────▼───────────────┐
│      Storage Layer       │   │      Protocol Layer      │
│     (src/archive.rs)     │   │     (whatsapp-rust)      │
│ - SQLite / SQLCipher     │   │ - Tokio WebSocket        │
│ - Encrypted local cache  │   │ - Protobuf serialization │
│ - Chats, messages, meta  │   │ - Noise protocol         │
│ - Key-value preferences  │   │ - AppState sync engine   │
└──────────────────────────┘   └──────────────────────────┘
```

---

## 1. Protocol Layer (`whatsapp-rust`) Interaction

ZapFast relies on `whatsapp-rust` as an external crate for WhatsApp Web protocol operations:
- **Session & Pairing:** Handles QR generation, pairing code authentication, companion device keys, and Noise handshakes.
- **Message Transport:** Maintains the secure WebSocket connection and manages frame encryption/decryption.
- **Sync Collections:** Replays initial history and continuous AppState sync collections (`RegularLow`, `RegularHigh`, `CriticalBlock`, `CriticalUnblock`).
- **Custom Protocol Extensions:**
  - WhatsApp settings (such as `setting_unarchiveChats`) arrive inside `RegularLow` AppState sync patches.
  - The custom fork (`YeralAndre/whatsapp-rust`) deserializes these actions and emits high-level events (e.g., `Event::UnarchiveChatsSettingUpdate`).

---

## 2. Backend & Worker (`src/backend/worker.rs`)

The `Worker` runs on a dedicated multi-threaded Tokio runtime:
- **Event Loop:** Receives protocol events from `whatsapp-rust` and UI commands from the interface channel.
- **In-Memory Cache:** Maintains fast runtime caches for account settings (e.g., `unarchive_chats: Option<bool>`), LID-to-phone mappings, and connection status.
- **Message Pipeline:**
  1. Ingests raw protobuf payloads and extracts attachments / media keys.
  2. Resolves privacy IDs (`@lid`) to canonical phone numbers (`@s.whatsapp.net`).
  3. Evaluates business logic (e.g., auto-unarchive checks upon receiving new incoming messages).
  4. Persists records to `Archive` and dispatches `Event::Messages` / `Event::ChatUpdated` to the UI thread.

---

## 3. Storage Layer (`src/archive.rs`)

Local data is stored in SQLite (encrypted via SQLCipher):
- **Core Entities:** `chats`, `messages`, `contacts`, `group_receipts`, `polls`, `stickers`, `drafts`.
- **Account Metadata & Preferences (`meta` table):**
  - Account-wide settings that do not belong to individual chats are stored as key-value pairs in `meta` (e.g., `key = "setting_unarchive_chats"`).
  - This avoids unnecessary schema migrations on the `chats` table for global preferences.
- **Conflict Resolution:**
  - Explicit user actions store millisecond timestamps (`archive_updated_at`, `lock_updated_at`).
  - Database updates verify timestamps to ensure older historical replays or delayed message arrivals do not override newer explicit actions.

---

## 4. UI Layer (`src/app.rs`, `src/ui/*`)

- **Immediate Mode:** Built on `egui` and `fastframe` for native multi-platform window management and rendering.
- **Unidirectional Data Flow:** Views render state provided by `App` and queue `model::Action`s. Views never directly mutate persistent database state.
