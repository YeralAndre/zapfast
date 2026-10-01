# Architecture Decision Records (ADRs)

This document records the key architectural and engineering decisions made in this repository.

---

## ADR 001: Persist Account-Wide Settings in Existing `meta` Table

### Context
ZapFast needs to persist the account-wide "Keep chats archived" (`unarchive_chats`) preference across application restarts and connection sessions.

### Decision
Store the preference in the existing `meta` key-value table:
- Key: `"setting_unarchive_chats"`
- Value: `"1"` for `true`, `"0"` for `false`, row absent for unset/unknown.

Do not add a column to the `chats` table, as this preference is global to the account, not per-chat.

### Consequences
- Eliminates schema migrations on the `chats` table.
- Encapsulates access cleanly inside `Archive::unarchive_chats_setting()` and `Archive::set_unarchive_chats_setting()`.
- Zero impact on chat query performance or table layout.

---

## ADR 002: Tri-State `Option<bool>` & Conservative Fallback for Unarchive Setting

### Context
When launching the app or before the first `RegularLow` AppState sync completes, the remote unarchive setting may not yet be known.

### Decision
Represent the runtime setting as `Option<bool>`:
- `Some(true)`: Unarchive enabled ("Keep chats archived" is OFF on phone) $\rightarrow$ incoming new message unarchives chat.
- `Some(false)`: Unarchive disabled ("Keep chats archived" is ON on phone) $\rightarrow$ chat remains archived.
- `None`: Unknown / not yet received from WhatsApp $\rightarrow$ **conservative fallback:** do NOT auto-unarchive.

### Consequences
- Prevents premature unarchiving during cold starts or offline historical replays before the user's preference is confirmed.

---

## ADR 003: Timestamp Guard on Chat Archive State (`archive_updated_at`)

### Context
Late-arriving incoming messages or replayed history could carry older timestamps than an explicit user archive/unarchive action performed on the desktop or phone.

### Decision
`Archive::set_archived_at` enforces timestamp comparison against `chats.archive_updated_at`. If an incoming event has a timestamp earlier than the latest explicit archive modification, the state change is ignored.

### Consequences
- Prevents out-of-order network packets or history replays from undoing user intent.

---

## ADR 004: Isolated Fix Branches + Personal Integration Branch

### Context
Multiple bug fixes (`empty-chat-rows`, `pinned-history-sync`, `archived-chat-sync`) are developed independently and require extensive daily testing before submitting upstream PRs.

### Decision
1. Maintain each fix on a dedicated, minimal `fix/*` branch.
   - `fix/archived-chat-sync` is confirmed rebased on `v0.18.1`.
   - `fix/empty-chat-rows` and `fix/pinned-history-sync` are isolated branches from earlier work whose rebase/audit onto `v0.18.1` is pending.
2. Audit and rebase all three fixes onto `v0.18.1`, then consolidate them into a separate personal integration branch for daily driving.
3. Keep `fix/*` branches clean and isolated for future upstream review.
4. Do not open upstream PRs until fixes have been verified in daily usage over several days.

### Consequences
- Independent, clean diffs ready for upstream review.
- No merge contamination or accidental coupling between unrelated fixes.

---

## ADR 005: Pin Custom Protocol Dependency by Commit SHA

### Context
ZapFast depends on features in `YeralAndre/whatsapp-rust` that are not yet merged upstream into `oxidezap/whatsapp-rust`.

### Decision
Specify the dependency in `Cargo.toml` using the exact Git revision hash (`rev = "80323773"`):
```toml
whatsapp-rust = { git = "https://github.com/YeralAndre/whatsapp-rust", rev = "80323773", default-features = false, features = ["sqlite-storage", "tokio-transport", "tokio-runtime", "ureq-client", "tokio-native"] }
```

### Consequences
- Deterministic selection of the whatsapp-rust source revision across machines and CI.
- Explicit control over protocol engine upgrades.

---

## ADR 006: Release & Audited Batch-Based Upstream and Dependency Synchronization

### Context
Upstream `crmne/zapfast` adopts trunk-based development with frequent commits, and protocol dependencies evolve continuously.

### Decision
1. Prefer upstream releases as natural milestones for synchronization and testing.
2. Update dependencies or pull upstream commits when ZapFast requires specific features or protocol/transport bug fixes.
3. Integrate small, clear batches of auditable commits as needed.
4. Require deeper architectural audit before merging large upstream refactors.

### Consequences
- Reduces unnecessary rebasing churn while remaining responsive to protocol and bugfix requirements.
- Ensures local testing occurs on stable, well-understood checkpoints.
