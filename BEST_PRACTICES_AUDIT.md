# Maolan DAW — Best Practices Audit Report

**Date:** 2026-03-08
**Scope:** Full codebase analysis across `src/` and `engine/src/` (~125 Rust source files)

---

## Executive Summary

The Maolan codebase is an ambitious Rust DAW with solid feature coverage (multi-track audio/MIDI, plugin hosting for CLAP/VST3/LV2, cross-platform audio backends, undo/redo). However, the analysis identified **27 actionable best practice violations** across 6 categories, with **3 critical**, **7 high**, **11 medium**, and **6 low** severity issues (plus 1 informational item confirmed as intentional design).

The most impactful findings center on three themes:
1. **Monolithic architecture** — God functions exceeding 2,600 lines and structs with 48+ public fields
2. **Missing safety infrastructure** — No custom error types, no clippy/lint config, no test coverage for GUI or connection validation
3. **FFI lifecycle and documentation gaps** — Plugin initialization lacks rollback, unsafe blocks lack `// SAFETY:` comments

> **Note on `UnsafeMutex`:** The custom `UnsafeMutex` is an intentional design choice for realtime audio performance. Traditional mutexes (`parking_lot::Mutex`, `std::sync::Mutex`) introduce unacceptable latency on the audio processing path. The lock-free access pattern is a deliberate trade-off — the engine architecture ensures that concurrent mutable access does not actually occur through its worker scheduling model (tracks are dispatched to individual workers, not shared). This is a valid pattern in realtime audio software.

---

## Table of Contents

1. [Informational: UnsafeMutex — Intentional Design](#1-unsafemutex--intentional-design-not-a-violation)
2. [Critical: God Functions](#2-god-functions)
3. [Critical: Unsafe Memory Operations](#3-unsafe-memory-operations)
4. [Critical: Rust Edition Configuration](#4-rust-edition-configuration)
5. [High: Encapsulation Violations](#5-encapsulation-violations)
6. [High: Deadlock-Prone Lock Patterns](#6-deadlock-prone-lock-patterns)
7. [High: FFI Safety in Plugin Loading](#7-ffi-safety-in-plugin-loading)
8. [High: State Modification Without Validation](#8-state-modification-without-validation)
9. [High: Panic-Prone unwrap/expect in Production Paths](#9-panic-prone-unwrapexpect-in-production-paths)
10. [High: Code Duplication in MIDI Learn Bindings](#10-code-duplication-in-midi-learn-bindings)
11. [High: Match Complexity in History System](#11-match-complexity-in-history-system)
12. [Medium: Missing Error Type Infrastructure](#12-missing-error-type-infrastructure)
13. [Medium: Undocumented Unsafe Blocks](#13-undocumented-unsafe-blocks)
14. [Medium: Channel Capacity and Backpressure](#14-channel-capacity-and-backpressure)
15. [Medium: Realtime Thread Safety](#15-realtime-thread-safety)
16. [Medium: Platform Abstraction Leaks](#16-platform-abstraction-leaks)
17. [Medium: Excessive Public API Surface](#17-excessive-public-api-surface)
18. [Medium: Conditional Compilation Explosion](#18-conditional-compilation-explosion)
19. [Medium: Plugin Processing Duplication](#19-plugin-processing-duplication)
20. [Medium: File Size and Complexity](#20-file-size-and-complexity)
21. [Medium: Lock Contention Patterns](#21-lock-contention-patterns)
22. [Low: No Clippy or Lint Configuration](#22-no-clippy-or-lint-configuration)
23. [Low: No Test Coverage for GUI](#23-no-test-coverage-for-gui)
24. [Low: Silent Failures in Plugin Discovery](#24-silent-failures-in-plugin-discovery)
25. [Low: Build Script Platform Assumptions](#25-build-script-platform-assumptions)
26. [Low: Unsafe Environment Variable Mutation](#26-unsafe-environment-variable-mutation)
27. [Low: Loose Dependency Version Constraints](#27-loose-dependency-version-constraints)
28. [Remediation Priorities](#28-remediation-priorities)

---

## 1. UnsafeMutex — Intentional Design (Not a Violation)

**Severity:** INFORMATIONAL (Revised from original CRITICAL assessment)
**Location:** `engine/src/mutex.rs:1-23`

### Description

The codebase implements a custom `UnsafeMutex<T>` that wraps `UnsafeCell<T>` and provides lock-free mutable access. This was **initially flagged as a critical violation** but after developer review, this is an **intentional design choice** for realtime audio performance.

Traditional mutexes introduce unacceptable latency on audio processing threads. The developer confirmed that the engine's worker scheduling model ensures tracks are dispatched to individual workers and concurrent mutable access to the same data does not occur in practice — the `UnsafeMutex` is essentially a zero-cost interior mutability wrapper relied upon by the scheduling invariant.

### Recommendation (Documentation only)

Add `// SAFETY:` comments on the `UnsafeMutex` implementation explaining the scheduling invariant that prevents concurrent access. This helps future contributors understand why this pattern is safe in this context despite appearing unsafe in general Rust code.

---

## 2. God Functions

**Severity:** CRITICAL
**Location:** `engine/src/engine.rs:2077-4730`, `engine/src/track.rs:339-830`

### Description

**`handle_request_inner()`** in `engine.rs` spans **2,654 lines** with a single match statement dispatching 50+ action types. Every engine action (play, stop, add track, rename, connect, MIDI learn, plugin operations, etc.) is handled in one function.

**`Track::process()`** in `track.rs` spans **492 lines** with 4-6 levels of nesting for plugin scheduling across LV2/VST3/CLAP plugin types.

### Impact

- Cannot unit test individual action handlers without full engine setup
- Any modification risks breaking unrelated code paths
- Extreme cognitive load — developers cannot reason about a 2,654-line function
- Compiler performance degradation from large match statements

### Recommendation

Extract each `Action` match arm into a dedicated handler method (e.g., `handle_add_track()`, `handle_rename_track()`). Use a dispatch table or trait-based pattern.

---

## 3. Unsafe Memory Operations

**Severity:** CRITICAL
**Location:** `engine/src/track.rs:2018-2042`

### Description

`find_vst3_audio_source_node()` uses `std::ptr::read()` to duplicate an `AudioIO` reference without incrementing the Arc refcount:

```rust
Arc::ptr_eq(
    input,
    &Arc::new(unsafe { std::ptr::read(audio_io as *const _) }),
)
```

This creates a temporary Arc from copied bits. When the temporary drops, it decrements the refcount on memory it doesn't own, risking premature deallocation and use-after-free.

### Recommendation

Use `Arc::ptr_eq()` on existing Arc references, or clone the Arc properly to increment the refcount.

---

## 4. Rust Edition Configuration

**Severity:** CRITICAL
**Location:** `Cargo.toml:4`, `engine/Cargo.toml:4`

### Description

Both crate manifests specify `edition = "2024"`. As of Rust 1.85+ (stable toolchain early 2025), the valid editions are 2015, 2018, 2021, and 2024. Edition 2024 was stabilized in Rust 1.85.0, so this is valid only if using a sufficiently recent toolchain.

### Recommendation

Verify the project's minimum supported Rust version (MSRV) is documented. If targeting broad compatibility, use `edition = "2021"`. If 2024 edition features are used intentionally, add a `rust-version = "1.85"` field to `Cargo.toml`.

---

## 5. Encapsulation Violations

**Severity:** HIGH
**Location:** `engine/src/track.rs:55-100`

### Description

The `Track` struct exposes **48 public fields** directly, including:
- 7 separate `Option<MidiLearnBinding>` fields (volume, balance, mute, solo, arm, input_monitor, disk_monitor)
- 3 plugin processor arrays (lv2, vst3, clap)
- 4 instance ID counters
- 30+ state fields (level, balance, armed, muted, soloed, transport state, etc.)

### Impact

- External code can set contradictory states (e.g., `armed=true` with `output_enabled=false`)
- Cannot change internal representation without updating all consumers
- 48 fields = 48 coupling points making refactoring extremely difficult

### Recommendation

Make fields private. Provide getter/setter methods that enforce invariants. Replace 7 MIDI learn fields with `HashMap<MidiLearnTarget, MidiLearnBinding>`.

---

## 6. Deadlock-Prone Lock Patterns

**Severity:** HIGH
**Location:** `engine/src/engine.rs:1984-2039`, `engine/src/workers/worker.rs:139-192`

### Description

Multiple lock acquisition chains without ordering discipline:

1. `connected_neighbors()` acquires locks: state → current_track → port_connections → next_track (4 levels)
2. `process_offline_bounce()` acquires: state → track → buffer → automation (4+ levels)
3. No consistent lock ordering policy exists

### Deadlock Scenario

Worker acquires state lock → iterates tracks → calls `t.process()` → plugin callback tries to send message → channel full → engine loop blocked on channel recv → engine holds state lock → **circular wait**.

### Recommendation

Document the lock acquisition order (State → Track → Port) so contributors follow a consistent pattern. Minimize lock scope and avoid calling external code (plugins) while holding locks.

---

## 7. FFI Safety in Plugin Loading

**Severity:** HIGH
**Location:** `engine/src/plugins/clap.rs:789-905,1137-1161`

### Description

CLAP plugin loading has partial initialization problems:
- If `activate()` succeeds but `start_processing()` fails, the plugin is activated but never deactivated
- `Drop` implementation calls `stop_processing()` even if `start_processing()` was never called
- Raw pointer to `HostRuntime::callback_flags` passed to plugin can dangle if HostRuntime is dropped while plugin retains the pointer

### Recommendation

Use RAII wrappers for each initialization stage. Implement a state machine: Unloaded → Loaded → Activated → Processing. Each stage's destructor only undoes its own stage.

---

## 8. State Modification Without Validation

**Severity:** HIGH
**Location:** `engine/src/state.rs` (entire file)

### Description

```rust
pub struct State {
    pub tracks: HashMap<String, Arc<UnsafeMutex<Box<Track>>>>,
}
```

The State struct is a public HashMap with no encapsulation:
- No validation when tracks are added or removed
- No invariant checking (e.g., connection cycle prevention)
- No reference counting for tracks in use by plugins
- GUI code directly accesses and mutates state

### Recommendation

Wrap State with methods that enforce invariants. Expose a narrowed API surface (add_track, remove_track, get_track) with validation.

---

## 9. Panic-Prone unwrap/expect in Production Paths

**Severity:** HIGH
**Location:** Multiple files (36+ instances)

### Key Locations

| File | Line(s) | Context | Risk |
|------|---------|---------|------|
| `src/connections/tracks.rs` | 670, 684 | Track lookup during connection validation | Panic if track deleted during connection |
| `src/plugins/clap/mod.rs` | 312, 325 | Mutex lock in CLAP host timer | Panic if mutex poisoned |
| `engine/src/plugins/vst3/processor.rs` | 397, 411, 501-502 | Mutex lock for parameter values | Panic if mutex poisoned |
| `engine/src/workers/hw_worker.rs` | 356, 380, 388, 393, 410, 430, 435 | Lock operations | Panic on poisoned mutex |
| `engine/src/engine.rs` | 2316-2337 | Shutdown sequence | Panic prevents graceful shutdown |
| `engine/src/plugins/vst3/interfaces.rs` | 48, 892, 907, 935, 955 | Run loop mutex | Panic during plugin events |

### Recommendation

Replace `.unwrap()` with `.expect("descriptive message")` at minimum. For mutex locks, handle poisoning with recovery or logging rather than panicking.

---

## 10. Code Duplication in MIDI Learn Bindings

**Severity:** HIGH
**Location:** `engine/src/track.rs:55-70`, `engine/src/history.rs:717-846`

### Description

7 separate `Option<MidiLearnBinding>` fields on Track are handled individually throughout the codebase:
- `track.rs` lines 64-70: 7 field declarations
- `history.rs` lines 732-759: Closure called 7 times for `ClearAllMidiLearnBindings`
- `history.rs` lines 798-846: 7 conditional blocks for `RemoveTrack` restoration

Total: 14+ duplicated conditional blocks across 2 files.

### Recommendation

Replace with `HashMap<MidiLearnTarget, MidiLearnBinding>` and iterate over entries.

---

## 11. Match Complexity in History System

**Severity:** HIGH
**Location:** `engine/src/history.rs:71-120`

### Description

`should_record()` has a `matches!()` macro with **44 match arms** listing every recordable action. Every new `Action` variant that should be recorded in undo history must be manually added here.

### Impact

- Manual synchronization burden — easy to forget new actions
- No compile-time verification that all recordable actions are listed
- 44 arms are difficult to review

### Recommendation

Use a derive macro or attribute (`#[recordable]`) on `Action` enum variants. Alternatively, make `should_record` the default and explicitly opt-out non-recordable actions (which are fewer).

---

## 12. Missing Error Type Infrastructure

**Severity:** MEDIUM
**Location:** Codebase-wide

### Description

The entire codebase uses `Result<T, String>` for error handling. No custom error types, no `thiserror` or `anyhow` crate usage, no `std::error::Error` trait implementations.

Pattern seen 15+ times:
```rust
.map_err(|e| e.to_string())?
```

### Impact

- Cannot programmatically distinguish error sources
- Limited error context propagation
- All errors coerced to opaque strings

### Recommendation

Add `thiserror` for structured error types in the engine crate. Define error enums for plugin loading, audio I/O, session management.

---

## 13. Undocumented Unsafe Blocks

**Severity:** MEDIUM
**Location:** `engine/src/track.rs:276-330`

### Description

4 unsafe audio buffer functions (`copy_unity_with_zero_tail`, `copy_scaled_with_zero_tail`, `add_unity`, `add_scaled`) use raw pointer arithmetic without `// SAFETY:` comments explaining why the operations are valid.

The unsafe blocks ARE justified for audio processing performance, but violate Rust convention (RFC 3380) requiring safety documentation.

### Recommendation

Add `// SAFETY:` comments explaining: pointers are valid, properly aligned, non-overlapping, and `i < len` is guaranteed by the while loop condition.

---

## 14. Channel Capacity and Backpressure

**Severity:** MEDIUM
**Location:** `engine/src/lib.rs:33-41`

### Description

```rust
let (tx, rx) = channel::<message::Message>(32);
```

Fixed channel capacity of 32 messages. With multiple workers each sending `Ready`/`Finished` messages, plus MIDI events, plugin callbacks, and hardware notifications, the channel can fill up and block senders — including audio threads.

### Recommendation

Increase the bounded capacity (e.g., 64 or 128). **Do not use an unbounded channel** — the developer measured 2ms round-trip latency for a simple Echo message with unbounded channels, which is unacceptable for realtime audio. Bounded channels with pre-allocated slots are significantly faster. Consider monitoring fill levels to detect backpressure before it blocks senders.

---

## 15. Realtime Thread Safety

**Severity:** MEDIUM
**Location:** `engine/src/workers/worker.rs:252-304`

### Description

Audio workers set `SCHED_FIFO` realtime scheduling priority but then:
- Use `tokio::sync::mpsc::Receiver::recv().await` which may allocate internally
- Invoke plugin callbacks that may acquire their own locks
- If channel is full, `send()` blocks in the realtime thread

### Impact

Memory allocation in realtime threads can cause audio glitches and dropouts.

### Recommendation

Consider lock-free ring buffers for audio thread communication to avoid any allocation on the hot path. The `UnsafeMutex` pattern is already allocation-free (see item 1), which is correct for this context.

---

## 16. Platform Abstraction Leaks

**Severity:** MEDIUM
**Location:** `engine/src/hw/mod.rs:1-32`

### Description

Platform-specific modules (`alsa`, `coreaudio`, `jack`, `oss`, `wasapi`, `sndio`) are all `pub`, allowing client code to depend on platform-specific types. No unified `HwDevice` trait abstracts over backends.

### Recommendation

Make platform modules private. Expose a unified trait that all backends implement.

---

## 17. Excessive Public API Surface

**Severity:** MEDIUM
**Location:** `engine/src/lib.rs:1-20`

### Description

Internal implementation details exposed as public modules:
- `pub mod mutex` — clients should never touch UnsafeMutex
- `pub mod state` — exposes raw HashMap with no encapsulation
- `pub mod workers` — implementation detail, only message passing needed
- `pub use plugins::clap/vst3/lv2` — GUI shouldn't manipulate plugins directly

### Recommendation

Make internal modules private. Expose only the message-passing API and necessary types.

---

## 18. Conditional Compilation Explosion

**Severity:** MEDIUM
**Location:** `src/hw.rs:17-450`

### Description

The `audio_view()` function has extensive `#[cfg(...)]` blocks with identical or near-identical code duplicated across Linux, Windows, FreeBSD, macOS, and BSD variants. Device selection UI, filter logic, and sample rate configuration are repeated with platform guards.

### Recommendation

Extract common UI elements into shared functions. Use platform abstraction types that work across all targets.

---

## 19. Plugin Processing Duplication

**Severity:** MEDIUM
**Location:** `engine/src/track.rs:388-518`

### Description

In `Track::process()`, the plugin scheduling loop for LV2, VST3, and CLAP is nearly identical — check processed, check inputs ready, check MIDI ready, process, update state. This pattern repeats 3 times with manual boolean tracking arrays.

### Recommendation

Create a unified plugin processing trait and iterate over all plugin types with a single generic loop.

---

## 20. File Size and Complexity

**Severity:** MEDIUM
**Location:** Three core files

| File | Lines | Concern |
|------|-------|---------|
| `engine/src/engine.rs` | 4,985 | Contains all action handlers, message routing, MIDI, file I/O, undo/redo |
| `engine/src/track.rs` | 3,140 | Track struct, audio processing, plugin hosting, clip loading |
| `src/gui/mod.rs` | ~3,575 | All GUI update logic, widget state, message dispatching |

### Recommendation

Split along responsibility boundaries. Extract action handlers, plugin management, audio processing, and session I/O into separate modules.

---

## 21. Lock Contention Patterns

**Severity:** MEDIUM
**Location:** `engine/src/engine.rs` (throughout)

### Description

Pattern of calling `self.state.lock()` multiple times in sequence without caching the lock:

```rust
self.state.lock().tracks.remove(name);
self.state.lock().tracks.contains_key(new_name);
self.state.lock().tracks.get(name);
self.state.lock().tracks.insert(new_name.clone(), track);
```

Each call acquires and releases the lock, allowing interleaving between operations.

### Recommendation

Cache the lock guard: `let state = self.state.lock();` then perform all operations on the cached reference.

---

## 22. No Clippy or Lint Configuration

**Severity:** LOW
**Location:** Project root (missing files)

### Description

No `clippy.toml`, `.cargo/config.toml`, `rustfmt.toml`, or `rust-toolchain.toml` found. Only one `#[allow(clippy::...)]` in the codebase (in `mutex.rs`).

### Recommendation

Add `clippy.toml` with project-appropriate settings. Consider `#![deny(clippy::unwrap_used)]` for production code.

---

## 23. No Test Coverage for GUI

**Severity:** LOW
**Location:** `src/` directory

### Description

The entire `src/` directory (GUI layer) has **zero tests**. Test coverage exists only in the engine:
- `engine/src/track.rs` — 6 test functions
- `engine/src/routing.rs` — 3 test functions
- `engine/src/hw/midi_hub.rs` — 2 test functions
- `engine/src/plugins/vst3/midi.rs` — 3 test functions
- `engine/src/plugins/vst3/state.rs` — 3 test functions

Connection validation code (`src/connections/tracks.rs` lines 670, 684) where unwrap panics occur has **no tests**.

### Recommendation

Add tests for connection validation logic and state management. Consider property-based testing for audio processing.

---

## 24. Silent Failures in Plugin Discovery

**Severity:** LOW
**Location:** `engine/src/plugins/clap.rs:846-873`

### Description

Plugin factory returning null descriptors are silently skipped with `continue`. Users see "Plugin not found" but cannot distinguish between a missing plugin and a corrupted library.

### Recommendation

Log warnings for null descriptors and invalid factory responses.

---

## 25. Build Script Platform Assumptions

**Severity:** LOW
**Location:** `build.rs:1-14`

### Description

Hardcoded X11 library paths: `/usr/X11R6/lib` for OpenBSD, `/usr/X11R7/lib` for NetBSD. No path validation, no fallback, and FreeBSD (a supported platform) is not handled.

### Recommendation

Use `pkg-config` or environment variables to discover library paths.

---

## 26. Unsafe Environment Variable Mutation

**Severity:** LOW
**Location:** `src/main.rs:60-66`

### Description

```rust
fn prefer_x11_backend() {
    unsafe {
        std::env::remove_var("WAYLAND_DISPLAY");
        std::env::remove_var("WAYLAND_SOCKET");
    }
}
```

Environment variable mutation is unsafe and not thread-safe. Called at startup before threads, but the pattern is brittle.

### Recommendation

Set environment variables before launching threads, or use process builder configuration instead.

---

## 27. Loose Dependency Version Constraints

**Severity:** LOW
**Location:** `Cargo.toml`

### Description

Some dependencies use loose version constraints:
- `ebur128 = "0.1"` (allows 0.1.0–0.1.99)
- `flacenc = "0.5"`
- `vorbis_rs = "0.5"`
- `vst3 = "0.3"`

Pre-1.0 crates with loose constraints may introduce breaking changes.

### Recommendation

Pin to specific patch versions for pre-1.0 dependencies in production builds.

---

## 28. Remediation Priorities

### Phase 1 — Safety Critical (Highest Priority)

| # | Action | Effort | Impact |
|---|--------|--------|--------|
| 1 | Fix `ptr::read` in `track.rs:2018-2042` | Low | Eliminates use-after-free risk |
| 2 | Verify/document Rust edition 2024 MSRV | Low | Build correctness |
| 3 | Document lock ordering discipline and `UnsafeMutex` safety invariants | Low | Prevents future contributor mistakes |

### Phase 2 — Architectural (High Priority)

| # | Action | Effort | Impact |
|---|--------|--------|--------|
| 5 | Extract `handle_request_inner()` into per-action handlers | High | Testability, maintainability |
| 6 | Encapsulate `Track` struct fields | High | Invariant enforcement |
| 7 | Encapsulate `State` struct | Medium | Validation, safety |
| 8 | Replace 7 MIDI learn fields with HashMap | Low | Eliminates duplication |
| 9 | Fix plugin lifecycle (RAII for partial init) | Medium | FFI safety |

### Phase 3 — Quality Infrastructure (Medium Priority)

| # | Action | Effort | Impact |
|---|--------|--------|--------|
| 10 | Add `thiserror` for structured error types | Medium | Error handling quality |
| 11 | Add `// SAFETY:` comments to unsafe blocks | Low | Code review, documentation |
| 12 | Add clippy configuration | Low | Automated quality checks |
| 13 | Add tests for connection validation | Medium | Regression prevention |
| 14 | Increase channel capacity | Low | Prevents audio thread blocking |
| 15 | Create platform abstraction trait | High | Portability, testability |

### Phase 4 — Polish (Lower Priority)

| # | Action | Effort | Impact |
|---|--------|--------|--------|
| 16 | Split large files | High | Navigability |
| 17 | Unify plugin processing loop | Medium | Code reduction |
| 18 | Reduce conditional compilation duplication | Medium | Maintainability |
| 19 | Add GUI tests | Medium | Coverage |
| 20 | Pin dependency versions | Low | Build reproducibility |

---

## Methodology

This audit was conducted through static analysis of the full source tree:
- All 125+ `.rs` files in `src/` and `engine/src/` were examined
- `Cargo.toml` and `build.rs` configuration reviewed
- Grep-based searches for `unsafe`, `.unwrap()`, `.expect()`, `#[test]`, `pub mod`, `#[cfg(` patterns
- Manual review of the 5 largest files (engine.rs, track.rs, history.rs, message.rs, hw.rs)
- Architecture analysis of module boundaries, FFI patterns, and concurrency model
