# LRU-based Garbage Collection Design

**Date:** 2026-02-01
**Topic:** Using registrationTime as an LRU signal for custom garbage collection

## Overview

The `updateRegistrationTime` feature is being implemented to support a smarter, custom garbage collection strategy that tracks path usage via registration time updates. This provides an LRU (Least Recently Used) garbage collection mechanism as an alternative to the traditional gcroots-only approach.

## Design Principle

**Only operational commands that actively use paths should update registration times.**

Registration time updates indicate: "This path was queried/used as part of an active operational decision"

## Commands That SHOULD Update Registration Times

These are commands that make operational decisions and actively use paths:
- **`nix build`** - Building derivations, evaluating outputs
- **`nix-build`** - Legacy build command
- **`nix eval`** - Evaluating Nix expressions
- **`nix-instantiate`** - Instantiating derivations from Nix code

**Implementation Status:** All four commands now call `.override(true)` to enable the setting.

## Commands That MUST NOT Update Registration Times

### Garbage Collection Commands
- **`nix store gc`** (new)
- **`nix-store --gc`** (legacy)

**Why:** These commands scan the *entire store* to determine what's garbage. If they updated registration times, they would mark every path as recently used, completely destroying the LRU signal.

The GC commands rely on gcroots to determine what's live. The registration time is kept as a separate data point for custom GC strategies that want to apply an LRU heuristic *in addition to* gcroots.

### Query-Only Commands
These should not update registration times:
- **`nix path-info`** - Just reading metadata
- **`nix verify`** - Checking store integrity (read-only)
- **`nix store info`** - Displaying store information
- **`nix query`** variants - Searching/displaying info

These are informational only and don't represent actual usage/operational decisions.

### Data Transfer Commands
- **`nix copy`** - Transfer mechanism, not a use of the path

Copying a path doesn't indicate it's being used—it could be archiving, backing up, or administrative movement.

## Custom GC Strategy

The intended use pattern:

1. **Nix commands execute normally**
   - `nix build`, `nix eval`, etc. update registration times
   - Tracks which paths have been actively used

2. **Custom GC script runs periodically**
   - Queries registrationTime from the Nix database
   - Identifies paths older than N days/weeks/months
   - Removes them (respecting gcroots)

3. **Benefits**
   - Traditional gcroots still respected (safety first)
   - LRU heuristic removes unused but reachable paths
   - More intelligent than just keeping everything with a gcroot
   - Complements rather than replaces the gcroots mechanism

## Existing Implementation

A working LRU GC implementation already exists in the `nix-csi` project:

```bash
nix path-info --store local --all --json | \
  jq -r --argjson age $seconds 'map(select(.registrationTime < (now - $age)) | .path) | .[]' | \
  nix store delete --store local --stdin --skip-live
```

**How it works:**
1. `nix path-info --all --json` - Query all store paths with metadata
2. `jq select(.registrationTime < (now - $age))` - Filter paths older than threshold
3. `nix store delete --stdin --skip-live` - Delete filtered paths, skipping anything still reachable by gcroots

The `--skip-live` flag ensures safety by respecting the existing gcroots mechanism.

## Implementation Checklist

- [x] Add `updateRegistrationTime` Setting to StoreConfig
- [x] Implement setting check in LocalStore::isValidPath_()
- [x] Enable in `nix build` command
- [x] Enable in `nix-build` command
- [x] Enable in `nix eval` command
- [x] Enable in `nix-instantiate` command
- [ ] **DO NOT** add to `nix store gc` or `nix-store --gc`
- [ ] **DO NOT** add to query-only commands

## Key Insight

**Registration time is metadata about command execution, not metadata about the store state.** Therefore:
- Commands that execute logic and make decisions → update times
- Commands that query state without making decisions → don't update times
- Commands that read all state (like GC) → definitely don't update times (would corrupt the signal)

## Future Work

Once this is implemented, a custom GC script could be written that:
1. Queries the Nix database for registrationTime values
2. Identifies paths older than a configured threshold
3. Safely deletes them while respecting gcroots
4. Could integrate with Nix's existing store.gc() or be a separate tool
