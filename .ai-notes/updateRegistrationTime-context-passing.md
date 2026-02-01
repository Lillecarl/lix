# updateRegistrationTime Context Passing Research

**Date:** 2026-02-01
**Topic:** Passing "updateRegistrationTime should be active" context from commands to store operations

## Problem Statement

Currently, `updateRegistrationTime()` is only called when commands are initiated over the daemon protocol. The goal is to make it work for both daemon and local store access from commands like:
- `nix-build`
- `nix-instantiate`
- `nix build`
- `nix eval`

The approach should be non-invasive and allow the updateRegistrationTime flag to "stick with the command."

## Key Findings

### 1. Thread-Local Storage Patterns (Viable)

Lix already uses thread-local variables:
- `curActivity` (libutil/logging.cc:22) - Tracks current Activity ID
- `AsyncContext::current` (libutil/async.hh:17) - Async context storage
- `interruptCheck` (libutil/signals.hh:44) - Interrupt checking callback

This shows thread-local storage is an acceptable pattern in Lix.

### 2. Activity System (Not Ideal for This Use Case)

The Activity/Logger system in libutil/logging.hh:
- Designed for progress tracking and logging hierarchies
- Not designed for carrying general execution context
- Has a stack via `PushActivity` but limited metadata capacity
- Would require invasive changes to repurpose

### 3. AsyncIoRoot (Viable but Invasive)

- Defined in libutil/async.hh
- Stored in thread-local `AsyncContext::current`
- Flows through command creation → store creation
- Could attach context but requires modifying Store API throughout

### 4. Current updateRegistrationTime Implementation

**Daemon** (libstore/daemon.cc):
- Called unconditionally after path queries (lines 281, 300, 849, 950)
- Works because daemon receives and deserializes paths over protocol

**LocalStore** (libstore/local-store.cc):
- Has SQL logic to update registration times in database
- Uses `UpdateRegistrationTimeRecursive` statement on store paths

**Problem:** No mechanism to signal local store access should call it

### 5. Command Execution Flow

```
main.cc → AsyncIoRoot created
  → NixArgs (MultiCommand)
    → StoreCommand subclass
      → createStore() [best insertion point]
        → openStore() returns Store implementation
          → isValidPath(), etc. called by command logic
```

**Best place to set context flag:** In `StoreCommand::createStore()` right after `openStore()` returns

### 6. StoreConfig Setting Pattern (Most Idiomatic)

Lix uses `Setting<T>` in StoreConfig for store-wide behavior:
- `readOnly` (LocalStoreConfig) - allow read-only databases
- `isTrusted` (StoreConfig) - trust level
- `systemFeatures` (StoreConfig) - supported features

This is the idiomatic way to control store behavior in Lix.

## Recommended Solution: StoreConfig Setting + StoreCommand Hook

### Why This Approach

✅ Uses existing Lix patterns (Settings in StoreConfig)
✅ Store-agnostic (works with any store implementation)
✅ Non-invasive (doesn't require modifying method signatures)
✅ Thread-safe (Settings designed for this)
✅ Configurable (can be set via CLI/config files)
✅ Daemon-compatible (daemon continues unchanged)

### Implementation Outline

**Step 1: Add Setting to StoreConfig** (libstore/store-api.hh)
```cpp
struct StoreConfig : public Config {
    // ... existing settings ...

    Setting<bool> updateRegistrationTime{this, false,
        "update-registration-time",
        "Auto-update registration times during path queries (for nix commands)."};
};
```

**Step 2: Check in updateRegistrationTime** (libstore/local-store.cc)
```cpp
kj::Promise<Result<bool>> LocalStore::updateRegistrationTime(const StorePathSet paths) {
    if (!config().updateRegistrationTime.get())
        co_return result::success(true);
    // ... actual implementation
}
```

**Step 3: Set Flag in StoreCommand** (libcmd/command.cc)
```cpp
ref<Store> StoreCommand::createStore() {
    auto store = aio().blockOn(openStore());
    store->config().updateRegistrationTime = true;  // Signal to update times
    return store;
}
```

**Step 4: No Changes Needed to isValidPath**
- isValidPath calls updateRegistrationTime unconditionally
- updateRegistrationTime checks the setting before doing work
- Separation of concerns: command signals intent, store checks feasibility

### Key Design Insight

**isValidPath doesn't need to know about command context.** It just calls `updateRegistrationTime()` unconditionally, and that method checks the setting. This avoids modifying the signature of isValidPath or adding context parameters throughout the store API.

## Alternative: Thread-Local Flag

For finer-grained per-operation control:

```cpp
// In execution-context.h
thread_local bool shouldUpdateRegistrationTime = false;

// In StoreCommand::run()
void StoreCommand::run(ref<Store> store) {
    shouldUpdateRegistrationTime = true;
    Finally cleanup([] { shouldUpdateRegistrationTime = false; });
    // ... operation
}

// In LocalStore::updateRegistrationTime()
if (!shouldUpdateRegistrationTime)
    co_return result::success(true);
```

**Pros:** Very lightweight, fine-grained control
**Cons:** Requires careful cleanup, less discoverable, not configurable

## Files to Modify

1. `libstore/store-api.hh` - Add Setting to StoreConfig, update virtual method
2. `libstore/local-store.cc` - Implement the setting check
3. `libcmd/command.cc` - Set flag in StoreCommand::createStore()
4. `libstore/remote-store.cc` - Optional: ensure RemoteStore respects it

## Why Not Other Approaches

| Approach | Issue |
|----------|-------|
| Modify isValidPath | Would require context parameters throughout; breaks abstraction |
| Use Activity metadata | Activity designed for logging, would require invasive changes |
| Attach to AsyncIoRoot | Would require passing context through all store operations |
| Global flag (no thread-local) | Race conditions in concurrent operations |
| Environment variable | Hacky, hard to test, not idiomatic |

## Two-Approach Strategy: Local + Daemon

After further research, the optimal solution is to use **two complementary approaches** that work independently without interference:

### Approach A: Local Command Invocation with StoreConfig Flag

For commands that directly create and use a LocalStore (e.g., `nix build` against local store):

1. **Add Setting to StoreConfig** (libstore/store-api.hh):
```cpp
Setting<bool> updateRegistrationTime{this, false,
    "update-registration-time",
    "Auto-update registration times during path queries (for nix commands)."};
```

2. **Check Setting in LocalStore** (libstore/local-store.cc):
```cpp
kj::Promise<Result<bool>> LocalStore::updateRegistrationTime(const StorePathSet paths) {
    if (!config().updateRegistrationTime.get())
        co_return result::success(true);
    // ... actual DB update with recursive SQL
}
```

3. **Set Flag in StoreCommand** (libcmd/command.cc):
```cpp
ref<Store> StoreCommand::createStore() {
    auto store = aio().blockOn(openStore());
    if (auto local = dynamic_cast<LocalStore *>(&*store)) {
        local->config().updateRegistrationTime = true;  // Enable for local stores
    }
    return store;
}
```

**Why this works:** StoreCommand is the base for all built-in Lix commands (CmdBuild, CmdEval, etc.). Setting the flag here automatically enables updateRegistrationTime for:
- nix build (local)
- nix-build (local)
- nix eval
- nix-instantiate
- All other StoreCommand subclasses

### Approach B: Daemon Protocol - Keep Existing Behavior

**Already implemented and working.** In daemon.cc (lines 281, 300, 849, 952):

```cpp
case WorkerProto::Op::IsValidPath: {
    bool result = aio.blockOn(store->isValidPath(path));
    to << result;
    aio.blockOn(store->updateRegistrationTime(path));  // ALWAYS called
    break;
}
```

The daemon unconditionally calls updateRegistrationTime after query operations. This ensures that:
- Any client connecting to the daemon gets updateRegistrationTime behavior
- Works even with unpatched Lix clients
- Respects daemon's own trust settings and storage decisions

**No changes needed to daemon.cc.**

### How They Work Together

**Different Code Paths - No Conflicts:**

```
When using local store directly:
  nix build (with --store=local or default)
  → StoreCommand::createStore()
  → Creates LocalStore, sets updateRegistrationTime = true
  → Command calls isValidPath(), QueryPathInfo(), etc.
  → LocalStore.updateRegistrationTime() runs (checks setting)
  → DB updated with recursive SQL [APPROACH A]

When using daemon store:
  nix build (with --store=daemon)
  OR
  any client connecting to daemon
  → StoreCommand::createStore()
  → Creates RemoteStore (no LocalStore on client side)
  → Client sends operation request through protocol
  → Daemon's processConnection() handles request
  → Daemon calls store->updateRegistrationTime() unconditionally
  → Daemon's LocalStore.updateRegistrationTime() runs
  → DB updated with recursive SQL [APPROACH B]
```

**Key insight:** RemoteStore doesn't implement updateRegistrationTime - it has the default no-op. The setting is only relevant for LocalStore instances.

### Why Settings Don't Travel Over Daemon Protocol

StoreConfig settings (like `updateRegistrationTime`) are **per-instance configuration**, not transmitted over the daemon protocol. Instead:
- The daemon has its own StoreConfig and settings
- Clients send ClientSettings (a small hardcoded struct) for a few specific settings
- Each side operates independently with its own configuration
- This is by design for security and isolation

### Integration Summary

| Context | Store Type | updateRegistrationTime Source |
|---------|-----------|------------------------------|
| Local command to local store | LocalStore | Approach A (StoreConfig flag) |
| Local command to daemon | RemoteStore | Approach B (daemon unconditional) |
| Remote client to daemon | RemoteStore | Approach B (daemon unconditional) |
| Unpatched client to patched daemon | RemoteStore | Approach B (daemon still works) |

### Why This Is Optimal

✅ **Non-invasive:** No changes to isValidPath or store operation signatures
✅ **Backward compatible:** Daemon behavior unchanged, purely additive
✅ **Works for both:** Local commands AND daemon connections
✅ **Idiomatic:** Uses existing Lix patterns (Settings, dynamic_cast)
✅ **Maintainable:** Clean separation of concerns
✅ **Secure:** Respects daemon protocol constraints
✅ **Future-proof:** Works with unpatched clients connecting to patched daemon

### Implementation Checklist

- [ ] Add `updateRegistrationTime` Setting to StoreConfig (store-api.hh)
- [ ] Implement setting check in LocalStore::updateRegistrationTime() (local-store.cc)
- [ ] Set flag in StoreCommand::createStore() for LocalStore instances (command.cc)
- [ ] Verify daemon.cc remains unchanged
- [ ] Test: `nix build` against local store
- [ ] Test: `nix build` against daemon
- [ ] Test: unpatched client to daemon

## Conclusion

**Use both approaches together** because:
1. Approach A handles local command invocation elegantly via StoreConfig
2. Approach B preserves daemon protocol functionality without modification
3. They operate in completely separate code paths and don't interfere
4. Together they cover all real-world usage scenarios

This design allows you to implement updateRegistrationTime for local commands while maintaining full compatibility with the existing daemon protocol, even for unpatched clients.
