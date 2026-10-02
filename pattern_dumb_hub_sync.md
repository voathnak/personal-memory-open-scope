---
name: pattern-dumb-hub-sync
description: "Dumb-Hub Sync — local-first multi-machine file sync over any dumb shared filesystem (NFS/SMB), coordinated by a client-side lease-fenced symlink lock; no smart server required"
aliases:
  - Dumb-Hub Sync
tags:
  - pattern
  - sync
created: 2026-07-19
---

# Dumb-Hub Sync

*Local-first, dumb-hub, lease-fenced sync.* A design pattern for keeping working copies of the
same file tree on several machines when all you have in the middle is a **dumb shared
filesystem** (NFS export, SMB share, even a USB disk) — no Dropbox-style smart server, no
per-file journal service.

This is my (Voathnak's) pattern, extracted from a working system (2026-07-19) built for AI-agent
memory vaults synced across a desktop, a single-board computer, and a NAS. Follow it freely;
nothing here is a mandate.

*Status (2026-10-02):* I replaced that system with plain git sync on 2026-08-16 once every
machine could reach a git host. The pattern still holds when all you have is a shared filesystem.

## The shape

- **Local-first:** every machine reads/writes only its own local copy. Full speed, full offline.
- **Hub-and-spoke:** each machine syncs *only* against the hub share. Machines never talk to each
  other, so the sync engine (unison) always runs single-machine against a mounted path — no
  cross-version RPC, no mesh.
- **Watch + tick:** a local filesystem watcher (reliable) triggers sync on change with a settle
  delay; a periodic tick (5 min) pulls remote changes and is the correctness floor. Never watch
  the hub — inotify/FSEvents don't propagate over network filesystems.
- **Newest-wins + conflict copies:** clock-gated (refuse to run unless NTP-synchronized), losers
  are preserved as conflict-copy files, never silently dropped.
- **Version history is the storage layer's job:** filesystem snapshots on the hub (btrfs/ZFS),
  not the sync layer.

## The hard part: coordination without a server

A dumb hub can't serialize writers, so the clients must. The primitive that survived adversarial
review (six rounds):

- **Atomic symlink lock:** acquire with `symlink(token, hub/.sync-ctl/lock)` — creation is
  atomic-and-exclusive on POSIX/NFS, and the owner identity lives *inside the link target*, so an
  uninitialized lock state cannot exist. Verify ownership by `readlink` after *every* attempt
  (network replies get lost).
- **Token-bound lease:** a lease file whose *content* is `<token> <timestamp>`, renewed by atomic
  replace. A lease only counts if its token matches the current lock. Renewal re-verifies lock
  ownership first — a deposed owner can never refresh a successor's lease.
- **Fenced takeover:** stale locks are quarantined by rename (never deleted), and the new owner
  waits one heartbeat-interval-plus-abort-grace *fence* before syncing — closing the window where
  a paused old owner could still be mid-write. Watchdog runtime < lease lifetime, enforced by
  config validation.
- **Control state lives outside the synced namespace** (`.sync-ctl/`, ignored by the engine).

## Other rules that earned their place

- Exclude `.git` from sync; repository history lives on exactly one designated machine, and git
  commands there run under the same local lock as the sync (torn working trees are real).
- Durable dirty-queue directory (atomic create to enqueue, batch-move to consume, retried on
  crash) unifies watcher and tick into one scheduler.
- Every non-zero engine exit is a failure; a staleness alarm fires when the last *verified*
  success is too old. Silent-stale is the failure mode that eats data months later.
- Snapshot the hub and prove one restore *before* first cutover, not after.

## When to use / not use

Use it when you own the machines, trust the LAN, and want auditability over convenience.
Use Syncthing/Dropbox/Synology Drive instead when you don't need custom scope boundaries,
per-file policy, or provable failure semantics — they're the same architecture with a smart
middle and far less code to own.
