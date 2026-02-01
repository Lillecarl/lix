# AI Research Notes Index

## Notes Files

### updateRegistrationTime Context Passing
**File:** `updateRegistrationTime-context-passing.md`

Complete research on implementing updateRegistrationTime for both local command invocation and daemon protocol access.

**Problem:**
- updateRegistrationTime currently only works over daemon protocol
- Need to extend it to local store access (nix-build, nix build, nix-instantiate, nix eval)
- Unpatched clients should still work with daemon

**Solution: Two-Approach Strategy**

**Approach A - Local Commands (StoreConfig Setting):**
- Add boolean `updateRegistrationTime` Setting to StoreConfig
- Check setting in LocalStore::updateRegistrationTime()
- Set flag in StoreCommand::createStore() for LocalStore instances
- Affects: nix-build, nix build, nix eval when used locally

**Approach B - Daemon Protocol (Keep Existing):**
- Leave daemon.cc unconditional updateRegistrationTime calls unchanged
- Works for all daemon connections, even unpatched clients
- No changes needed

**Key Findings:**
- Store settings DON'T travel over daemon protocol (by design)
- RemoteStore uses ClientSettings (hardcoded subset), not StoreConfig
- Two approaches operate on completely different code paths
- No interference or conflicts between approaches
- Using `dynamic_cast<LocalStore *>()` to apply Approach A only to LocalStore instances

**Related Files:**
- libstore/store-api.hh (add Setting to StoreConfig)
- libstore/local-store.cc (check setting in updateRegistrationTime)
- libcmd/command.cc (set flag in StoreCommand::createStore)
- libstore/daemon.cc (UNCHANGED - keep current behavior)
- libstore/remote-store.cc (reference only)

---

### LRU-based Garbage Collection Design
**File:** `lru-gc-design.md`

Design and rationale for using registrationTime as an LRU signal for custom garbage collection.

**Use Case:**
- Traditional Lix GC relies only on gcroots to determine what's garbage
- This feature enables a smarter, LRU-based GC that identifies unused but reachable paths
- Respects gcroots while adding intelligent removal of old, unused paths

**Key Design Principle:**
Only operational commands that actively use/query paths should update registration times. Query and administrative commands must not, especially GC commands which would destroy the LRU signal.

**Commands That Update Times:**
- nix build, nix-build, nix eval, nix-instantiate

**Commands That Must NOT Update Times:**
- nix store gc, nix-store --gc (would corrupt LRU signal)
- nix path-info, nix verify, nix query (query-only)
- nix copy (data transfer, not usage)

---

**Last Updated:** 2026-02-01
