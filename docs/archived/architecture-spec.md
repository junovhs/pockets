# ClipboardPower — Final Product Specification

**Version:** v0.0.1
**Date:** January 2025
**Author:** Spencer + Claude
**Purpose:** This document is the complete specification for ClipboardPower. It will be handed to an AI coding assistant to implement. Every decision, edge case, and design rationale is included.

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [The Problem We're Solving](#2-the-problem-were-solving)
3. [Target Users](#3-target-users)
4. [Product Positioning](#4-product-positioning)
5. [The Core Workflow](#5-the-core-workflow)
6. [Interaction Model (Complete)](#6-interaction-model-complete)
7. [Visual Design (Complete)](#7-visual-design-complete)
8. [Security](#8-security)
9. [Technical Architecture](#9-technical-architecture)
10. [Design Philosophy](#10-design-philosophy)
11. [What's NOT in v0.0.1](#11-whats-not-in-v0.0.1)
12. [Edge Cases & Error Handling](#12-edge-cases--error-handling)
13. [Open Questions](#13-open-questions)
14. [Launch Plan](#14-launch-plan)
15. [Success Metrics](#15-success-metrics)

---

## 1. Executive Summary

**Name:** ClipboardPower

**Domain:** clipboardpower.com

**One-liner:** Copy in order. Paste in order. Done.

**Price:** $12 one-time purchase

**Platform:** Windows (v0.0.1)

**What it is:** A batch transfer tool for data entry workers. NOT a clipboard manager.

**The core mechanic:**
- Hold Ctrl+C → enter queue mode
- Copy A, B, C → queue fills (📋 1, 📋 2, 📋 3)
- Ctrl+V, Ctrl+V, Ctrl+V → pastes A, B, C in order
- Queue empties → back to normal

**The killer feature:** Select 4 cells in Excel, hold Ctrl+C, and each cell becomes its own slot. One copy action loads 4 items.

---

## 2. The Problem We're Solving

### The Status Quo

The clipboard hasn't changed since 1974. Copy replaces what you had. One slot, no memory, constant overwriting.

### The Pain (Quantified)

A data entry worker copying 4 fields from a spreadsheet to a web form:

**Today:**
1. Left screen: click cell, Ctrl+C
2. Right screen: click field, Ctrl+V
3. Left screen: click cell, Ctrl+C
4. Right screen: click field, Ctrl+V
5. Left screen: click cell, Ctrl+C
6. Right screen: click field, Ctrl+V
7. Left screen: click cell, Ctrl+C
8. Right screen: click field, Ctrl+V

**8 screen jumps per record.** 50 records per day = 400 screen jumps.

**With ClipboardPower:**
1. Left screen: select 4 cells, hold Ctrl+C
2. Right screen: Ctrl+V, Ctrl+V, Ctrl+V, Ctrl+V

**1 screen jump per record.** 50 records = 50 screen jumps.

**87% reduction in context switching.**

### The Emotional Pain

- **Overwrite disaster:** You cut something important, copy something else, it's gone forever
- **Clipboard anxiety:** "Did I copy the right thing? Did I paste it already?"
- **Cognitive load:** Mentally tracking which field you're on
- **Fatigue:** The mechanical repetition of Alt-Tab, click, Ctrl+C, Alt-Tab, click, Ctrl+V, repeat 400 times

### Why Existing Tools Don't Solve This

| Tool | Problem |
|------|---------|
| Win+V | LIFO order (backwards), requires popup + click every paste |
| Ditto | Complex config, ugly UI, still requires selecting each paste |
| CopyQ | 80mb+, steep learning curve, overkill |
| Keyboard Maestro | Requires building macros, not for non-technical users |

**The gap:** No tool offers FIFO sequential paste with zero UI per paste.

---

## 3. Target Users

### Primary: Data Entry Workers

**Who they are:**
- Admins and coordinators
- HR specialists
- Support reps
- Accountants
- Executive assistants
- Medical records clerks
- Anyone moving data between systems

**Their day:**
- Copy fields from spreadsheets, PDFs, emails
- Paste into CRMs, web forms, internal systems
- Dozens to hundreds of times per day

**Their technical level:**
- Not developers
- Don't know about Vim registers or macros
- Can learn one new gesture, not a complex system

**Real validation (Spencer's coworker):**
> "As a Data Entry People, I fully agree!"
> "ALL the agents do a lot of copy/pasting"
> "They will always use the data given by clients for sensitive info rather than re-typing"

### Secondary: Anyone Doing Batch Transfer

- Researchers collecting quotes from multiple sources
- Writers restructuring documents
- QA testers entering test data
- Anyone who copies multiple things to paste elsewhere

---

## 4. Product Positioning

### We Are NOT a Clipboard Manager

Clipboard managers (Win+V, Ditto, CopyQ) are **random-access databases**:
- Copy things over time
- Browse history
- Pick what you want each time you paste
- Good for: "I need that thing I copied 3 hours ago"

ClipboardPower is a **sequential stream** (FIFO):
- Copy multiple things in a batch
- Paste them in order
- No browsing, no picking per paste
- Good for: "I need to move these 5 fields from here to there"

### The Positioning Statement

> ClipboardPower is a batch transfer tool for people who move data between systems. Copy in order. Paste in order. Done.

### Competitive Comparison

| | Win+V | Ditto | ClipboardPower |
|---|-------|-------|----------------|
| Order | LIFO | LIFO | **FIFO** |
| Sequential paste | No | No | **Yes** |
| Random access | Click popup | Click popup | Number keys |
| Multi-cell split | No | No | **Yes** |
| UI per paste | Popup + click | Popup + click | **None** |
| Stays open | No | Configurable | **Yes (pinnable)** |
| Learning curve | Low | High | **Very low** |
| Price | Free | Free | $12 |
| Target | General | Power users | **Data entry** |

---

## 5. The Core Workflow

### Workflow A: Multiple Copies (Field by Field)

**User's situation:** Copying 4 fields from a PDF into a web form.

1. **Enter queue mode:** Hold Ctrl+C for 500ms
   - Visual: 🔴 appears near cursor

2. **Build the queue:**
   - Select title, tap Ctrl+C → 📋 1
   - Select description, tap Ctrl+C → 📋 2
   - Select date, tap Ctrl+C → 📋 3
   - Select price, tap Ctrl+C → 📋 4

3. **Switch to destination:** Alt-Tab to web form (one time)

4. **Paste sequentially:**
   - Click title field, Ctrl+V → pastes title, icon shows 📋 3
   - Click description field, Ctrl+V → pastes description, icon shows 📋 2
   - Click date field, Ctrl+V → pastes date, icon shows 📋 1
   - Click price field, Ctrl+V → pastes price, icon disappears

5. **Done:** Queue empty, back to normal clipboard behavior

### Workflow B: Single Copy of Multiple Cells (Excel Power Move)

**User's situation:** Copying a row of 4 cells from Excel into a web form.

1. **Select multiple cells:** Click and drag across 4 cells in Excel

2. **Enter queue mode + copy:** Hold Ctrl+C for 500ms
   - Clipboard receives: `"Title\tDescription\tDate\tPrice"`
   - App detects tabs, splits into 4 slots
   - **Floating window auto-opens** showing all 4 items with previews
   - Cursor icon shows 📋 4

3. **Switch to destination:** Alt-Tab to web form

4. **Paste sequentially:** Same as Workflow A

**Critical:** When multi-cell is detected, the floating window MUST auto-open. This makes the state crystal clear. User immediately sees "oh, it split my data into 4 slots."

### Workflow C: Random Access (Need to Paste Out of Order)

**User's situation:** Has 4 items queued but needs to paste slot 3 first.

1. **Open floating window:** Click tray icon (or it's already open from multi-cell)

2. **See the slots:**
   ```
   ┌─────────────────────────────────────┐
   │ ClipboardPower            📌  ✕    │
   ├─────────────────────────────────────┤
   │ 1: "Summer Sale"                   │
   │ 2: "support@example.com"           │
   │ 3: "2025-06-30"                    │
   │ 4: "$500"                          │
   └─────────────────────────────────────┘
   ```

3. **Press number key:** Press "3" on keyboard
   - Slot 3 visually "presses" (button feedback)
   - "2025-06-30" pastes into the active field
   - Slot 3 remains in window (not consumed)

4. **Can paste same slot multiple times:** Press "3" again → pastes again

5. **Resume sequential:** Ctrl+V still works for FIFO discharge

**Critical rule:** Number keys ONLY work when the floating window is OPEN. This prevents the nightmare scenario of typing "2" in a form and accidentally pasting.

---

## 6. Interaction Model (Complete)

### 6.1 Entering Queue Mode

**Trigger:** Hold Ctrl+C for 500ms

**What happens technically:**
1. User presses Ctrl+C
2. Normal Ctrl+C fires IMMEDIATELY (we do not block or delay)
3. We start a 500ms timer
4. If user releases before 500ms → nothing special, normal copy happened
5. If user holds past 500ms → queue mode activates

**What the user sees:**
- Immediate: Normal copy happens (selection goes to clipboard)
- After 500ms: 🔴 red "recording" dot appears near cursor
- System tray icon changes to indicate queue mode

**Why this is safe:**
- We never block Ctrl+C
- Tap behavior is identical to native (sacred)
- Hold adds capability without changing default

### 6.2 Building the Queue (Individual Copies)

**Trigger:** Ctrl+C tap while in queue mode

**What happens technically:**
1. User presses Ctrl+C (tap)
2. Normal copy happens (OS puts content in clipboard)
3. We read clipboard content
4. We add content to our queue (`VecDeque<String>`)
5. We update visual

**What the user sees:**
- 🔴 changes to 📋 1 (first item)
- Next copy: 📋 2
- Brief preview flashes (first ~20 chars, 1-2 seconds)

**Technical note:** During the copy phase, we do NOT modify the system clipboard. We only read from it. The clipboard contains whatever the user last copied.

### 6.3 Building the Queue (Multi-Cell Detection)

**Trigger:** Ctrl+C while multiple cells are selected (Excel, Google Sheets, etc.)

**What happens technically:**
1. User selects multiple cells
2. User presses Ctrl+C (or hold Ctrl+C to enter queue mode)
3. OS puts tab-separated values in clipboard: `"A\tB\tC\tD"`
4. We detect tabs (or newlines) in clipboard content
5. We split into multiple slots
6. We **auto-open the floating window**

**Splitting logic:**
```
If content contains tabs AND newlines (grid):
  → Split by lines, then by tabs, flatten (row by row, left to right)
  
If content contains tabs only (horizontal row):
  → Split by tabs
  
If content contains newlines only (vertical column):
  → Split by newlines
  
Otherwise (single item):
  → One slot
```

**What the user sees:**
- Floating window opens automatically
- All slots visible with previews
- Cursor icon shows 📋 N (total count)
- State is CRYSTAL CLEAR

**Why auto-open the window:**
This is critical for user understanding. If someone selects 4 cells and presses Ctrl+C, they need to immediately see that 4 separate items were captured. Without the window, they might not realize the split happened.

### 6.4 Sequential Paste (FIFO Discharge)

**Trigger:** Ctrl+V while queue is active

**What happens technically (The Post-Paste Swap):**

1. **First Ctrl+V in queue mode:**
   - Before paste: We inject `queue[0]` into system clipboard
   - User's Ctrl+V keypress goes through to OS
   - OS pastes from clipboard (our injected content)

2. **We detect Ctrl+V KeyUp event**

3. **Wait 50ms** (safety buffer for slow apps like Electron)

4. **Inject next item:**
   - Remove `queue[0]` (it was just pasted)
   - If queue not empty: inject `queue[0]` (new front) into clipboard
   - Update visual: 📋 3 → 📋 2

5. **Repeat until empty**

6. **When empty:**
   - Exit queue mode
   - Cursor icon disappears
   - Normal clipboard behavior resumes

**Why this works:**
- We're not intercepting Ctrl+V
- We're not racing before the paste
- We swap the clipboard AFTER the paste completes
- If app crashes mid-paste, Ctrl+V still works normally

**The 50ms safety buffer:**
Some apps (especially Electron-based: Slack, VS Code, Discord) are slow to read the clipboard. If we swap instantly after KeyUp, they might read the NEW value instead of what was intended. 50ms gives them time to complete the read.

### 6.5 Random Access (Number Keys)

**Trigger:** Press number key (1-9) while floating window is OPEN

**What happens:**
1. User presses "3"
2. We check: is floating window open? (MUST BE YES)
3. Slot 3 visually "presses" (button animation)
4. Content from slot 3 is pasted (we write to clipboard, simulate Ctrl+V)
5. Slot 3 **remains** in the queue (non-consuming)

**Critical rule: Number keys ONLY work when window is OPEN**

| Window State | Number Key Behavior |
|--------------|---------------------|
| Closed | Types the number normally ("2" appears in text field) |
| Open | Pastes from that slot |

**Why this rule exists:**

Imagine: You're in queue mode, cursor icon shows 📋 3, but you're typing in a form. You need to enter "Room 201". You type "Room 2"... and suddenly it pastes something random.

**RAGE. UNINSTALL.**

By requiring the window to be open, we make the mode obvious:
- "If you can see the slots, you can press their numbers"
- Close the window = keyboard works normally
- No invisible modes, no surprises

### 6.6 Click to Paste (Alternative Random Access)

**Trigger:** Click a slot in the floating window

**What happens:**
1. User clicks slot 2
2. Content from slot 2 pastes to wherever cursor/focus was
3. Slot 2 remains (non-consuming)

**Same as number keys, but with mouse.**

### 6.7 Exiting Queue Mode

**Method 1: Natural completion**
- Paste until queue is empty
- Icon disappears
- Automatically exits queue mode

**Method 2: Escape key**
- Press Escape
- Queue clears
- Icon disappears
- Exits queue mode

**Method 3: Close floating window**
- Click X on floating window
- Queue clears
- Icon disappears
- Exits queue mode

**Method 4: Hold Ctrl+C again**
- Hold Ctrl+C for 500ms (same gesture as entering)
- Queue clears
- Stays in queue mode (ready for new copies)
- Use case: "Start over, I messed up"

---

## 7. Visual Design (Complete)

### 7.1 Cursor Icon (Status Only)

**Purpose:** At-a-glance status while working. NOT interactive.

**States:**

| State | Visual | Meaning |
|-------|--------|---------|
| Not in queue mode | (nothing) | Normal clipboard |
| Queue mode, ready | 🔴 | Recording, waiting for copies |
| Queue has 1 item | 📋 1 | 1 item ready to paste |
| Queue has 3 items | 📋 3 | 3 items ready |
| After 1 paste (was 3) | 📋 2 | 2 items remaining |
| Queue empty | (disappears) | Back to normal |

**Behavior:**
- Follows cursor (offset to upper-right, ~20px away)
- Non-interactive (cannot click, cannot hover for info)
- Small but visible (~24x24px)
- Does NOT change keyboard behavior (that's the window's job)

**Visual style:**
- Clean, minimal
- Consistent with Windows 11 aesthetic
- High contrast (visible on any background)

### 7.2 Floating Window (Interactive)

**Purpose:** Detailed view, random access, management

**How to open:**
- Click system tray icon
- Auto-opens on multi-cell detection
- Optional: user-configurable hotkey

**Layout:**

```
┌─────────────────────────────────────────┐
│ ClipboardPower                  📌  ✕  │
├─────────────────────────────────────────┤
│ [1] "Summer Sale - 50% off all..."     │
│ [2] "support@example.com"              │
│ [3] "2025-06-30"                       │
│ [4] "$500"                             │
└─────────────────────────────────────────┘
```

**Elements:**

| Element | Behavior |
|---------|----------|
| Title bar | Draggable |
| 📌 Pin button | Toggle always-on-top |
| ✕ Close button | Clears queue, exits queue mode |
| Slot rows | Clickable (pastes content) |
| Slot numbers | Correspond to keyboard numbers |

**Slot preview:**
- Shows first ~30 characters
- Truncates with "..." if longer
- Monospace or proportional font (test both)

**Number key feedback:**
When user presses "2":
- Slot [2] visually "presses" (background color darkens, slight scale)
- Quick animation (100ms)
- Satisfying, game-like feel

**Pin behavior:**
- 📌 gray = not pinned (window closes on focus loss)
- 📌 colored = pinned (window stays on top)
- Pinned window stays visible even when typing in other apps

**Window size:**
- Width: ~350px (fits reasonable previews)
- Height: Dynamic based on slot count
- Max slots visible before scroll: ~8-10
- Scrollable if more

### 7.3 System Tray Icon

**States:**

| State | Icon | Tooltip |
|-------|------|---------|
| Running, idle | Gray clipboard | "ClipboardPower" |
| Queue mode active | Green clipboard + badge | "Queue: 3 items" |
| Sensitive data detected | Orange clipboard | "Queue: 3 items (sensitive)" |

**Click:** Opens floating window

**Right-click menu:**
```
├── Open ClipboardPower
├── Clear Queue
├── ─────────────
├── Settings
├── ─────────────
├── Exit
```

---

## 8. Security

### 8.1 Core Principles

1. **Memory only** — Queue is NEVER written to disk. No database. No file. No cache.

2. **No cloud** — Nothing leaves the machine. Ever.

3. **No telemetry** — We don't track anything. No analytics. No crash reports sent home.

4. **Auto-clear** — Queue clears on:
   - App exit
   - Queue emptied by pasting
   - Escape pressed
   - Window closed

5. **Minimal surface** — Small Rust binary. No Electron. No webview. Minimal dependencies.

### 8.2 Sensitive Data Detection

**Patterns to detect:**

| Type | Pattern |
|------|---------|
| Credit card | 16 digits (with or without spaces/dashes) |
| SSN | XXX-XX-XXXX |
| API key | Common prefixes (sk_, pk_, api_, etc.) |
| Password | From password manager markers |

**When detected:**

1. **Visual warning:**
   - Slot highlighted orange/red in floating window
   - Cursor icon changes color
   - Tray icon changes color

2. **Behavior remains the same:**
   - Still pastes normally
   - Still in queue
   - v0.0.1: Warning only, no blocking

### 8.3 Password Manager Integration

**Honor OS-level markers:**
- Windows: `CF_EXCLUDECLIPBOARDHISTORY` clipboard format

**If marker is present:**
- Do NOT add to queue
- Ignore this copy
- Let it exist only in native clipboard

**Why:** Password managers like 1Password and Bitwarden set these markers. We respect them.

---

## 9. Technical Architecture

### 9.1 Stack

| Component | Choice | Why |
|-----------|--------|-----|
| Language | Rust | Memory safety, small binary, fast |
| Clipboard | `arboard` crate | Cross-platform, reliable |
| Global hotkeys | `global-hotkey` or `rdev` | Non-blocking listening |
| Input simulation | `enigo` | Cross-platform key simulation |
| System tray | `tray-icon` | Native tray integration |
| Window | `tao` + custom rendering | Lightweight, no webview |

### 9.2 Memory & Size Targets

| Metric | Target |
|--------|--------|
| Binary size | < 5MB |
| RAM usage (idle) | < 10MB |
| RAM usage (full queue) | < 20MB |
| Startup time | < 500ms |

**No Electron. No webview. No Chromium.**

### 9.3 State Machine

```
                    ┌─────────────────────────────────────────┐
                    │                                         │
                    ▼                                         │
┌─────────┐   Hold Ctrl+C    ┌─────────────┐                 │
│  IDLE   │ ────────────────▶│ QUEUE_MODE  │                 │
└─────────┘     (500ms)      └─────────────┘                 │
     ▲                              │                         │
     │                              ├── Ctrl+C: add to queue  │
     │                              │                         │
     │                              ├── Ctrl+V: paste, advance│
     │                              │                         │
     │                              ├── Number key (window    │
     │                              │   open): paste slot     │
     │                              │                         │
     │   ┌──────────────────────────┤                         │
     │   │                          │                         │
     │   │  Queue empty             │  Escape                 │
     │   │  after paste             │  Close window           │
     │   │                          │                         │
     └───┴──────────────────────────┴─────────────────────────┘
```

### 9.4 Core Data Structures

```rust
use std::collections::VecDeque;

struct AppState {
    mode: Mode,
    queue: VecDeque<QueueItem>,
    window_open: bool,
    window_pinned: bool,
}

enum Mode {
    Idle,
    QueueMode,
}

struct QueueItem {
    content: String,
    is_sensitive: bool,
    timestamp: Instant,
}
```

### 9.5 Key Algorithms

**Hold Detection:**
```rust
fn on_ctrl_c_down(&mut self) {
    // Native Ctrl+C fires immediately (we don't block)
    self.ctrl_c_down_time = Some(Instant::now());
}

fn on_ctrl_c_up(&mut self) {
    if let Some(down_time) = self.ctrl_c_down_time {
        if down_time.elapsed() >= Duration::from_millis(500) {
            // Long press detected
            if self.mode == Mode::Idle {
                self.enter_queue_mode();
            } else {
                // Already in queue mode - clear and restart
                self.clear_queue();
            }
        }
        // Short press: normal copy happened, we'll capture it
        if self.mode == Mode::QueueMode {
            self.capture_clipboard();
        }
    }
    self.ctrl_c_down_time = None;
}
```

**Multi-Cell Parsing:**
```rust
fn parse_clipboard_content(content: &str) -> Vec<String> {
    let has_tabs = content.contains('\t');
    let has_newlines = content.contains('\n');
    
    let items: Vec<String> = if has_tabs && has_newlines {
        // Grid: split by lines, then tabs, flatten (row by row)
        content
            .lines()
            .flat_map(|line| line.split('\t'))
            .map(|s| s.trim().to_string())
            .filter(|s| !s.is_empty())
            .collect()
    } else if has_tabs {
        // Horizontal row
        content.split('\t')
            .map(|s| s.trim().to_string())
            .filter(|s| !s.is_empty())
            .collect()
    } else if has_newlines {
        // Vertical column
        content.lines()
            .map(|s| s.trim().to_string())
            .filter(|s| !s.is_empty())
            .collect()
    } else {
        // Single item
        vec![content.to_string()]
    };
    
    items
}

fn capture_clipboard(&mut self) {
    let clipboard = Clipboard::new().unwrap();
    if let Ok(content) = clipboard.get_text() {
        let items = parse_clipboard_content(&content);
        
        let multi_cell = items.len() > 1;
        
        for item in items {
            let is_sensitive = detect_sensitive(&item);
            self.queue.push_back(QueueItem {
                content: item,
                is_sensitive,
                timestamp: Instant::now(),
            });
        }
        
        // AUTO-OPEN window on multi-cell detection
        if multi_cell {
            self.open_window();
        }
        
        self.update_visuals();
    }
}
```

**Post-Paste Swap:**
```rust
fn on_ctrl_v_released(&mut self) {
    if self.mode != Mode::QueueMode || self.queue.is_empty() {
        return;
    }
    
    // Remove the item that was just pasted
    self.queue.pop_front();
    
    if self.queue.is_empty() {
        self.exit_queue_mode();
        return;
    }
    
    // Safety buffer for slow apps (Electron, etc.)
    thread::sleep(Duration::from_millis(50));
    
    // Load next item into clipboard
    if let Some(next) = self.queue.front() {
        let mut clipboard = Clipboard::new().unwrap();
        clipboard.set_text(&next.content).unwrap();
    }
    
    self.update_visuals();
}
```

**Number Key Handling:**
```rust
fn on_number_key(&mut self, num: usize) {
    // CRITICAL: Only works when window is open
    if !self.window_open {
        // Let the keypress through as normal typing
        return;
    }
    
    if num == 0 || num > self.queue.len() {
        return;
    }
    
    let index = num - 1; // 1-indexed to 0-indexed
    if let Some(item) = self.queue.get(index) {
        // Visual feedback
        self.animate_slot_press(num);
        
        // Write to clipboard
        let mut clipboard = Clipboard::new().unwrap();
        clipboard.set_text(&item.content).unwrap();
        
        // Simulate Ctrl+V
        let mut enigo = Enigo::new();
        enigo.key_down(Key::Control);
        enigo.key_click(Key::Layout('v'));
        enigo.key_up(Key::Control);
        
        // Note: We do NOT remove from queue (non-consuming)
    }
}
```

**Sensitive Data Detection:**
```rust
fn detect_sensitive(content: &str) -> bool {
    // Credit card (16 digits)
    let cc_pattern = Regex::new(r"\b\d{4}[\s-]?\d{4}[\s-]?\d{4}[\s-]?\d{4}\b").unwrap();
    if cc_pattern.is_match(content) {
        return true;
    }
    
    // SSN (XXX-XX-XXXX)
    let ssn_pattern = Regex::new(r"\b\d{3}-\d{2}-\d{4}\b").unwrap();
    if ssn_pattern.is_match(content) {
        return true;
    }
    
    // API keys (common prefixes)
    let api_prefixes = ["sk_", "pk_", "api_", "key_", "token_"];
    for prefix in api_prefixes {
        if content.to_lowercase().contains(prefix) {
            return true;
        }
    }
    
    false
}
```

---

## 10. Design Philosophy

### The Guiding Principle

> Intuit, be delighted at your validated intuition — and when you're wrong, it's a nice learning experience that never happens again because it's a simple thing you learned without trying.

### Principles

1. **Ctrl+C and Ctrl+V are sacred**
   - Tap behavior is IDENTICAL to native
   - We never block, delay, or modify taps
   - Hold adds capability without changing default

2. **State is always visible**
   - Cursor icon shows queue status
   - Window shows slot contents
   - Tray icon shows mode
   - Never guess what mode you're in

3. **If you can see it, you can press it**
   - Number keys work only when window is open
   - No invisible keyboard modes
   - No surprises

4. **Escape always works**
   - Press Escape = clear queue, exit mode
   - Universal, expected, reliable

5. **Progressive disclosure**
   - Basic: Hold Ctrl+C, copy, paste
   - Advanced: Open window for previews
   - Power: Number keys for random access
   - Each level is optional

6. **No configuration required**
   - Works out of the box
   - Settings exist but aren't needed
   - Sensible defaults

7. **Crash safety**
   - If app crashes, clipboard works normally
   - No corrupted state
   - No data loss (queue was temporary anyway)

### What "Game-Like" Means

- **Visual feedback:** Icons animate, slots "press," count updates
- **Satisfying:** Each action has clear response
- **Discoverable:** Try things, see what happens
- **Low stakes:** Can't break anything, easy to undo
- **Flow state:** Gets out of the way once learned

---

## 11. What's NOT in v0.0.1

| Feature | Why Not |
|---------|---------|
| Auto-advance (Tab/Enter after paste) | User said table it; clicking between fields is common |
| Persistent slots / snippets | Different feature (saved templates vs temporary queue) |
| Cloud sync | Security risk; not needed for target users |
| Strip formatting / clean paste | Not our problem; stay focused |
| History / search | We're a batch transfer tool, not a database |
| Mac / Linux | Windows first; cross-platform later |
| Images / files | Text only in v0.0.1; files are complex |
| Plugins / scripting | Way out of scope |

**The constraint is the feature.** We do one thing well: FIFO batch transfer.

---

## 12. Edge Cases & Error Handling

### Edge Case: Copy Same Content Twice

**Scenario:** User copies "Hello" twice in a row.

**Behavior:** Two slots, both containing "Hello". FIFO still works.

**Why:** User might need to paste "Hello" twice. We don't deduplicate.

### Edge Case: Copy Empty Selection

**Scenario:** User tries to copy with nothing selected.

**Behavior:** Clipboard is empty or unchanged. We don't add empty slots.

### Edge Case: Very Long Content

**Scenario:** User copies 10,000 words.

**Behavior:**
- Slot preview truncates to ~30 chars
- Full content stored in queue
- Pastes correctly

### Edge Case: Multi-Cell with Empty Cells

**Scenario:** User selects `"A\t\tC"` (middle cell empty).

**Behavior:** Split into ["A", "C"]. Empty strings are filtered out.

**Alternative:** Could keep empty as a slot. Decide during testing.

### Edge Case: Number Key with No Queue

**Scenario:** Window is open but queue is empty. User presses "1".

**Behavior:** Nothing happens. (Or show brief "Queue is empty" indicator?)

### Edge Case: Paste When Queue Empty

**Scenario:** User presses Ctrl+V but queue is already empty.

**Behavior:** Normal paste (whatever is in system clipboard).

### Edge Case: App Crash Mid-Queue

**Scenario:** App crashes with 3 items in queue.

**Behavior:**
- Queue is lost (was only in memory)
- Ctrl+V works normally (pastes whatever was last in system clipboard)
- No corruption, no broken state
- User restarts app, starts fresh

### Edge Case: Window Open but Queue Empty

**Scenario:** User opens window but hasn't copied anything yet.

**Behavior:** Window shows "Queue is empty" message. Number keys do nothing.

### Edge Case: Extremely Fast Pasting

**Scenario:** User mashes Ctrl+V very quickly.

**Behavior:**
- Post-paste swap has 50ms buffer
- If user pastes faster than buffer, they might get same item twice
- Acceptable trade-off vs. blocking input

### Edge Case: Conflicting Hotkey

**Scenario:** Another app uses the same global hotkey.

**Behavior:**
- We lose the race (or both fire)
- User can change our hotkey in settings
- Ctrl+C/V are OS-level, we don't compete there (we observe, not intercept)

---

## 13. Open Questions

These need answers during development/testing:

1. **Preview flash duration:** When copying, preview flashes briefly. How long? 1 second? 2 seconds? Test with users.

2. **Max queue size:** Should there be a limit? 10? 50? 100? Unlimited? Memory concern vs. user freedom.

3. **Sound feedback:** Subtle sounds on:
   - Entering queue mode?
   - Each copy?
   - Each paste?
   - Queue empty?
   
   Test if this is delightful or annoying.

4. **Default hotkey for window:** What opens the floating window?
   - Click tray icon (definitely)
   - Keyboard shortcut? Which one?

5. **Multi-cell edge: Tab in content:** What if a cell contains a literal tab character? Currently would split incorrectly. Rare but possible. Excel might escape it?

6. **Window position:** Where does floating window appear?
   - Near cursor?
   - Center screen?
   - Remember last position?

7. **Escape behavior:** Should Escape close window only, or also clear queue? Currently: clears queue and exits. Is that too destructive?

---

## 14. Launch Plan

### Phase 1: Core (Week 1-2)
- [ ] Global hotkey listening (Ctrl+C hold detection)
- [ ] Queue data structure
- [ ] Clipboard reading
- [ ] Multi-cell parsing (tab/newline splitting)
- [ ] Post-paste clipboard swap
- [ ] Cursor icon (basic: dot + number)
- [ ] System tray icon

**Milestone:** Can hold Ctrl+C, copy 3 things, Ctrl+V pastes them in order.

### Phase 2: Window (Week 3-4)
- [ ] Floating window framework
- [ ] Slot list with previews
- [ ] Pin functionality
- [ ] Number key paste (when window open)
- [ ] Click-to-paste
- [ ] Visual feedback (slot press animation)
- [ ] Auto-open on multi-cell detection

**Milestone:** Full interaction model working.

### Phase 3: Polish (Week 5-6)
- [ ] Sensitive data detection + visual warning
- [ ] Password manager marker check
- [ ] Settings panel (hotkey config, etc.)
- [ ] Installer
- [ ] Code signing
- [ ] Icon design (professional)

**Milestone:** Shippable product.

### Phase 4: Launch (Week 7)
- [ ] Website (simple landing page)
- [ ] Payment integration (Gumroad? Paddle? Stripe?)
- [ ] Share with coworker for testing
- [ ] Fix feedback
- [ ] Soft launch
- [ ] Iterate

---

## 15. Success Metrics

| Metric | Target | Why |
|--------|--------|-----|
| Cold start | < 500ms | Must feel instant |
| Memory (idle) | < 10MB | Lightweight |
| Memory (full queue) | < 20MB | Still lightweight |
| Added paste latency | < 10ms | Imperceptible |
| Binary size | < 5MB | Easy download |
| Window switches saved | 80%+ | The whole point |
| Refund rate | < 5% | Product works |

---

## Appendix A: Terminology

| Term | Definition |
|------|------------|
| Queue mode | Active state where copies accumulate and pastes discharge in order |
| Slot | One item in the queue |
| FIFO | First In, First Out (copy A,B,C → paste A,B,C) |
| LIFO | Last In, First Out (what Win+V does — backwards) |
| Sequential paste | Ctrl+V automatically advances to next slot |
| Random access | Paste specific slot by number or click |
| Discharge | Paste and remove from queue (Ctrl+V) |
| Non-consuming | Paste but keep in queue (number keys / click) |
| Multi-cell | Copying multiple cells at once (tab/newline separated) |
| Post-paste swap | Loading next item into clipboard AFTER paste completes |

---

## Appendix B: Keyboard Reference

| Key | Window Closed | Window Open |
|-----|---------------|-------------|
| Hold Ctrl+C | Enter queue mode (or clear if already in) | Same |
| Tap Ctrl+C | Add to queue | Same |
| Ctrl+V | Paste + advance (sequential) | Same |
| 1-9 | Type number normally | Paste from slot (non-consuming) |
| Escape | Clear queue, exit mode | Same |
| Click slot | N/A | Paste from slot (non-consuming) |

---

## Appendix C: Visual States Summary

**Cursor Icon:**
| State | Icon |
|-------|------|
| Normal (no queue mode) | (nothing) |
| Queue mode, empty | 🔴 |
| Queue mode, has items | 📋 N |

**Tray Icon:**
| State | Icon |
|-------|------|
| Idle | Gray clipboard |
| Queue mode active | Green clipboard + badge |
| Sensitive detected | Orange clipboard |

**Floating Window:**
| State | Appearance |
|-------|------------|
| Closed | Not visible |
| Open, empty | "Queue is empty" message |
| Open, has items | Slot list with previews |
| Pinned | 📌 highlighted, stays on top |

---

## Appendix D: This Is NOT a Clipboard Manager

If anyone asks "how is this different from Win+V / Ditto / CopyQ?", the answer is:

**Those are databases. This is a stream.**

| Clipboard Manager | ClipboardPower |
|-------------------|----------------|
| Random access (pick each time) | Sequential (paste in order) |
| LIFO (last copied = first shown) | FIFO (first copied = first pasted) |
| Popup every paste | No popup (Ctrl+V just flows) |
| "Find that thing from earlier" | "Move these 5 fields right now" |
| History over time | Temporary batch |
| Power users, developers | Data entry workers |

**We are a batch transfer tool.**

---

*End of specification.*
