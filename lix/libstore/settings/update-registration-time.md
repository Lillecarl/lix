---
name: update-registration-time
internalName: updateRegistrationTime
type: bool
default: false
---
Auto-update registration times during path queries in operational commands.

When enabled, the store will update the `registrationTime` field of store paths
when they are queried during operational commands like `nix build`, `nix eval`,
and `nix-instantiate`. This enables LRU-based garbage collection where paths that
are never built or evaluated have their registration times left unchanged,
allowing custom GC scripts to identify and remove unused but reachable paths.

This setting is typically only enabled by operational commands, not by query or
GC commands, to preserve the signal for LRU collection.
