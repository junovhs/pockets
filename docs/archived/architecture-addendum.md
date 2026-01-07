# architecture-addendum.md
## ClipboardPower — Architecture Addendum (Carry-Over Essentials)

**Status:** Addendum to v0.0.1 Final Product Specification  
**Applies to:** ClipboardPower (Windows v0.0.1)  
**Purpose:** Capture the critical engineering guardrails and low-level mechanics that must be implemented (or explicitly designed around) to ensure correctness, reliability, and “sacred shortcut” behavior.

---

## 1) Non-Negotiable Principles

### 1.1 Sacred Shortcuts (Tap Must Be Native)
**Rule:** Tap interactions for `Ctrl+C` and `Ctrl+V` must remain indistinguishable from native OS behavior.

**Implications:**
- No added latency that is perceptible (target <10ms added for the tap path).
- No blocking of the OS handler for the tap path.
- Any “enhanced behavior” must be introduced only via:
  - Hold gesture detection, or
  - Post-event clipboard swaps that occur after the OS has already completed the user’s intended action.

**Acceptance test:** A user who never holds keys or opens the window must never notice ClipboardPower exists.

---

## 2) Clipboard Write Echo Suppression (Critical Correctness)

### 2.1 Problem
ClipboardPower writes to the system clipboard during:
- Sequential paste injection and post-paste swapping
- Random-access paste via number keys
- Click-to-paste in the floating window

These writes can trigger clipboard change notifications and/or be re-read by our own capture logic. Without suppression, you risk:
- Re-capturing our own injected content
- Duplicating entries
- Corrupting queue ordering
- Creating feedback loops in some event models

### 2.2 Requirement
**Requirement:** Every clipboard capture must verify that the observed clipboard content is not a recent self-write.

### 2.3 Minimal Design
Maintain a lightweight record of the last clipboard write performed by ClipboardPower:
- Content hash (fast, non-cryptographic is acceptable)
- Timestamp
- Optional: monotonic “write sequence id” for debugging

**Echo rule:** If a read matches the last self-write hash and occurs within a short time window, treat it as an echo and ignore it for queue capture.

### 2.4 Recommended Parameters
- `ECHO_WINDOW_MS`: 250–500ms (default 300ms)

### 2.5 Pseudocode
```rust
struct LastClipboardWrite {
    hash: u64,
    at: Instant,
}

fn should_ignore_as_echo(last: Option<&LastClipboardWrite>, content: &str) -> bool {
    let h = fast_hash(content);
    match last {
        Some(lw) => lw.hash == h && lw.at.elapsed() < Duration::from_millis(ECHO_WINDOW_MS),
        None => false,
    }
}
````

**Implementation note:** Apply echo suppression in `capture_clipboard()` (or equivalent) before enqueuing.

---

## 3) Clipboard Locked / Access Failures (Retry & Backoff)

### 3.1 Problem

On Windows, clipboard operations frequently fail transiently because another process holds the clipboard open (Excel, RDP, Electron apps, password managers, etc.). This is normal OS behavior.

If clipboard reads/writes are assumed to always succeed, the product will appear “randomly broken.”

### 3.2 Requirement

**Requirement:** All clipboard reads and writes must be resilient to transient lock failures via bounded retries and short backoff.

### 3.3 Recommended Strategy

* Retry a small number of times (e.g., 5 attempts)
* Backoff in small increments (e.g., 10ms, 20ms, 30ms, …)
* If still failing:

  * For reads: treat as “no capture” (do not crash)
  * For writes during paste: avoid consuming the queue item if we could not successfully stage the clipboard

### 3.4 Recommended Parameters

* `CLIPBOARD_RETRY_COUNT`: 5
* `CLIPBOARD_RETRY_BASE_DELAY_MS`: 10 (linear backoff is sufficient)

### 3.5 Pseudocode

```rust
fn read_clipboard_text_with_retry() -> Result<String, ClipboardError> {
    let mut last_err = None;
    for i in 0..CLIPBOARD_RETRY_COUNT {
        match try_read_clipboard_text_once() {
            Ok(s) => return Ok(s),
            Err(e) => last_err = Some(e),
        }
        std::thread::sleep(Duration::from_millis(CLIPBOARD_RETRY_BASE_DELAY_MS * (i as u64 + 1)));
    }
    Err(last_err.unwrap_or(ClipboardError::Unknown))
}
```

---

## 4) Clipboard Timing Reality: Delayed Rendering & Slow Readers

### 4.1 Problem

Many applications do not read the clipboard instantaneously (notably Electron apps). If ClipboardPower swaps the clipboard too quickly after a paste, the target app may read the “next” value, resulting in incorrect pasted data.

ClipboardPower already uses a post-paste delay buffer; this must be treated as an intentional compatibility mechanism.

### 4.2 Requirements

1. **Post-paste delay must be configurable** (even if not exposed in UI for v0.0.1).
2. ClipboardPower must not assume that a KeyUp event guarantees the target application has already read the clipboard.
3. If reliability is threatened, prefer correctness over maximum throughput.

### 4.3 Recommended Parameters

* `POST_PASTE_DELAY_MS`: default 50ms
* Allow tuning up to 150ms (practical ceiling before users notice lag)

### 4.4 Operational Guidance

* Keep the delay constant initially.
* If field reports indicate incorrect pastes on specific apps, increase this delay or implement per-process overrides (future option).

---

## 5) Failure Taxonomy & Behavioral Contracts

### 5.1 Purpose

Engineers must treat common failure modes as expected and handle them without user confusion or crashes.

### 5.2 Categorization & Expected Behavior

#### A) Expected / Transient Errors (Handle Silently)

* Clipboard locked
* Clipboard empty
* Temporary read/write failures

**Behavior:**

* Retry with backoff
* If still failing: no crash; skip capture or skip injection safely

#### B) Recoverable Errors (Degrade Gracefully)

* UI render failure
* Tray icon failure
* Window open/close races
* Input simulation not available momentarily

**Behavior:**

* Clipboard queue logic remains functional even if UI is impaired
* Attempt re-init on next user action where safe

#### C) Unrecoverable Errors (Fail Fast with Clear Diagnostics)

* Internal invariant violations (e.g., impossible state machine transition)
* Persistent corruption of in-memory structures (should be rare)

**Behavior:**

* Fail with clear log output
* Do not corrupt OS clipboard behavior on exit

### 5.3 Key Contract: Do Not Consume on Failed Injection

For sequential paste and random-access paste:

* If ClipboardPower cannot successfully set clipboard text, it must not “advance” / pop from queue.
* The queue remains intact until a successful staging + paste cycle occurs.

---

## 6) Minimal Observability (Local-Only, No Telemetry)

### 6.1 Goal

Diagnose real-world edge cases without violating the “no telemetry” principle.

### 6.2 Requirements

* Local logging only (file or Windows event log), off by default or low-volume by default
* No network calls
* No clipboard contents written to logs (redact or hash)

### 6.3 Suggested Logged Events (No Content)

* Mode transitions (Idle ↔ QueueMode)
* Queue length changes
* Clipboard read/write failures (error type, retry count)
* Post-paste timing used (ms)
* Echo suppression triggered (count only)

---

## 7) Platform Constraints (Document Now to Prevent Future Drift)

### 7.1 Purpose

Even with Windows-only v0.0.1, future platform plans must respect OS realities. These constraints should be documented to avoid misleading roadmaps or accidental architectural decisions.

### 7.2 Non-Negotiables for Future Ports

* **macOS:** Global keyboard monitoring typically requires Accessibility permissions; distribution constraints differ from Windows.
* **Linux Wayland:** True global hotkey capture is intentionally restricted; solutions vary by compositor and may require portals or privileged access.
* **Clipboard models differ:** Ownership semantics and eventing differ substantially across platforms.

### 7.3 Required Documentation Artifact

Maintain a short `PLATFORM_CONSTRAINTS.md` (internal is fine) that states:

* What is straightforward
* What is permission-gated
* What is fundamentally constrained by platform security models

---

## 8) Summary Checklist (Engineering Gate)

Before considering v0.0.1 “done,” verify:

* [ ] Echo suppression implemented and tested
* [ ] Clipboard read/write retry with backoff implemented; no panics in clipboard path
* [ ] Post-paste delay exists and can be tuned without code surgery
* [ ] Queue does not advance on failed injection
* [ ] Tap-path behavior remains native-feeling
* [ ] UI failure does not break core copy/paste mechanics
* [ ] Logging does not record clipboard contents; remains local-only
* [ ] Platform constraints documented to prevent roadmap drift

---

**End of addendum**
