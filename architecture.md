# Pockets: Technical Architecture Proposal

**Status:** Approved Specification  
**Version:** 0.1.0  
**Last Updated:** 2025-12-30  
**Author:** Architecture Team

---

## Executive Summary

Pockets is a cross-platform clipboard augmentation layer that provides multiple clipboard slots with zero learning curve. It observes the system clipboard without modification, serves content to it on demand, and presents a minimal near-cursor UI that appears contextually during copy/paste operations.

**The Core Insight:** The system clipboard is sacred. We don't replace it—we feed it.

---

## Table of Contents

1. [Design Philosophy](#1-design-philosophy)
2. [The Three Laws of Pockets](#2-the-three-laws-of-pockets)
3. [Interaction Model](#3-interaction-model)
4. [System Architecture](#4-system-architecture)
5. [Platform Abstraction Layer](#5-platform-abstraction-layer)
6. [Clipboard Engine](#6-clipboard-engine)
7. [Hotkey Interception System](#7-hotkey-interception-system)
8. [UI Layer](#8-ui-layer)
9. [Storage & Persistence](#9-storage--persistence)
10. [Security Model](#10-security-model)
11. [Performance Requirements](#11-performance-requirements)
12. [Platform-Specific Considerations](#12-platform-specific-considerations)
13. [Error Handling Philosophy](#13-error-handling-philosophy)
14. [Testing Strategy](#14-testing-strategy)
15. [Distribution & Packaging](#15-distribution--packaging)
16. [Future Considerations](#16-future-considerations)

---

## 1. Design Philosophy

### 1.1 The Grandma Test

Every feature must pass the Grandma Test: Can you explain it in one sentence using physical metaphors?

**Pockets:** "Extra pockets for your clipboard. Copy more, lose nothing."

If a feature requires technical explanation, it's too complex.

### 1.2 The Injection Model

Modern game modding teaches us: don't modify the game files, inject at runtime.

Pockets follows the same principle:
- We **never** modify system clipboard behavior
- We **never** hook into OS internals
- We **observe** clipboard changes via public APIs
- We **serve** content to the clipboard when requested
- The OS and applications are unaware of our existence

### 1.3 Progressive Disclosure

- **Zero-config users:** Ctrl+C and Ctrl+V work exactly as before
- **Curious users:** Notice the icons, explore naturally
- **Power users:** Use hold-gestures and keyboard shortcuts

No tutorials. No onboarding. The UI teaches by appearing.

### 1.4 Symmetry Principle

Copy and Paste are mirrors. Every interaction pattern available for Copy must exist for Paste:

| Copy Action | Paste Action |
|-------------|--------------|
| Tap Ctrl+C → copies to active | Tap Ctrl+V → pastes from active |
| Tap + click slot → copies there | Tap + click slot → pastes from there |
| Hold 1.5s → picker appears | Hold 1.5s → picker appears |

No exceptions. No special cases.

---

## 2. The Three Laws of Pockets

### Law 1: The Law of Sanctity (Sacred Shortcuts)

**Constraint:** Ctrl+C and Ctrl+V behavior MUST be indistinguishable from native behavior for tap interactions.

**Rationale:** These shortcuts are 40-year-old muscle memory. They are cultural. They are sacred. A user who never discovers Pockets' extra features must never notice Pockets exists.

**Enforcement:** 
- Tap latency must be <10ms added to native
- Default slot always receives/provides content
- No confirmation dialogs, no delays, no interception on tap

### Law 2: The Law of Visibility (The Clipboard Is Real)

**Constraint:** The current clipboard state MUST always be knowable without opening any window.

**Rationale:** The original clipboard metaphor (a physical board with a clip) was visible. Digital clipboards broke this. We restore it.

**Enforcement:**
- System tray icon reflects active slot state
- Near-cursor UI appears on every copy/paste (briefly)
- Hover reveals content preview
- Active slot is always visually distinct (green)

### Law 3: The Law of Resilience (Never Lose Data)

**Constraint:** User data in slots MUST survive: app crashes, system sleep, reboots, and updates.

**Rationale:** A clipboard manager that loses clipboard history is worse than no clipboard manager.

**Enforcement:**
- Write-ahead logging for slot changes
- Atomic file operations only
- Crash recovery on startup
- Content hashing for integrity verification

---

## 3. Interaction Model

### 3.1 Core Concepts

```
┌─────────────────────────────────────────────────────────────┐
│                    POCKETS MENTAL MODEL                     │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   SYSTEM CLIPBOARD          POCKETS SLOTS                   │
│   ┌───────────────┐        ┌───┬───┬───┬───┬───┐           │
│   │               │◄──────►│ 1 │ 2 │ 3 │ 4 │ 5 │           │
│   │  (The Stage)  │        └───┴───┴───┴───┴───┘           │
│   │               │         ▲                               │
│   └───────────────┘         │                               │
│          ▲                  │ "Active" slot                 │
│          │                  │ (highlighted green)           │
│          │                  │                               │
│     [Ctrl+V reads]     [Pockets writes to                  │
│     [Ctrl+C writes]     system clipboard                   │
│                         when slot selected]                 │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Key Insight:** The system clipboard is always the "stage." Pockets slots are backstage. When you select a slot, we copy its contents TO the system clipboard. Then normal Ctrl+V pastes from system clipboard as always.

### 3.2 Interaction Matrix

| Gesture | Timing | Behavior |
|---------|--------|----------|
| Ctrl+C | Tap (<300ms) | Copy selection to active slot AND system clipboard. Show slot icons for 2s. |
| Ctrl+C | Tap + Click slot | Copy selection to clicked slot. Make it active. |
| Ctrl+C | Tap + Press 1-9 | Copy selection to numbered slot. Make it active. |
| Ctrl+C | Hold (≥1.5s) | Show slot picker. Selection pending. Press 1-9 or click to copy there. |
| Ctrl+V | Tap (<300ms) | Paste from system clipboard (sacred). Show slot icons for 2s. |
| Ctrl+V | Tap + Click slot | Load slot to system clipboard. Paste. Make it active. |
| Ctrl+V | Tap + Press 1-9 | Load numbered slot to system clipboard. Paste. Make it active. |
| Ctrl+V | Hold (≥1.5s) | Show slot picker. Press 1-9 or click to paste from that slot. |

### 3.3 State Transitions

```
                    ┌──────────────┐
                    │    IDLE      │
                    │ (listening)  │
                    └──────┬───────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
              ▼            ▼            ▼
        ┌──────────┐ ┌──────────┐ ┌──────────┐
        │  COPY    │ │  PASTE   │ │   TRAY   │
        │ DETECTED │ │ DETECTED │ │  CLICK   │
        └────┬─────┘ └────┬─────┘ └────┬─────┘
             │            │            │
             ▼            ▼            │
        ┌──────────┐ ┌──────────┐     │
        │  SHOW    │ │  SHOW    │     │
        │  ICONS   │ │  ICONS   │     │
        │ (2s TTL) │ │ (2s TTL) │     │
        └────┬─────┘ └────┬─────┘     │
             │            │            │
    ┌────────┴────────────┴────────────┘
    │
    ▼
┌─────────────────────────────────────────┐
│              ICON STATE                  │
│  ┌─────┐ ┌─────┐ ┌─────┐                │
│  │  1  │ │  2  │ │ ... │                │
│  │(grn)│ │(wht)│ │(gry)│                │
│  └─────┘ └─────┘ └─────┘                │
│                                          │
│  • Green = Active slot                   │
│  • White = Has content                   │
│  • Gray = Empty                          │
│                                          │
│  TRANSITIONS:                            │
│  • Mouse hover → extend TTL, show preview│
│  • Click slot → select, action completes │
│  • Press 1-9 → select, action completes  │
│  • TTL expires → fade out → IDLE         │
│  • Escape → cancel → IDLE                │
└─────────────────────────────────────────┘
```

### 3.4 Timing Parameters

| Parameter | Default | Range | Notes |
|-----------|---------|-------|-------|
| `tap_threshold_ms` | 300 | 100-500 | Max duration to count as "tap" |
| `hold_threshold_ms` | 1500 | 1000-3000 | Duration before hold-mode activates |
| `icon_display_ms` | 2000 | 1000-5000 | How long icons remain visible |
| `icon_fade_ms` | 300 | 100-500 | Fade-out animation duration |
| `hover_preview_delay_ms` | 200 | 0-500 | Delay before showing preview on hover |

All timing parameters are configurable. Defaults are hypotheses to be validated through user testing.

---

## 4. System Architecture

### 4.1 High-Level Component Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                              POCKETS                                     │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                      APPLICATION LAYER                           │    │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐           │    │
│  │  │   UI Layer   │  │  Slot Mgr    │  │  Config Mgr  │           │    │
│  │  │   (Dioxus)   │  │              │  │              │           │    │
│  │  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘           │    │
│  │         │                 │                 │                    │    │
│  │         └─────────────────┼─────────────────┘                    │    │
│  │                           │                                       │    │
│  │                    ┌──────▼───────┐                              │    │
│  │                    │  Event Bus   │                              │    │
│  │                    │  (channels)  │                              │    │
│  │                    └──────┬───────┘                              │    │
│  │                           │                                       │    │
│  └───────────────────────────┼───────────────────────────────────────┘    │
│                              │                                            │
│  ┌───────────────────────────┼───────────────────────────────────────┐    │
│  │                    CORE LAYER                                      │    │
│  │         ┌─────────────────┼─────────────────┐                     │    │
│  │         │                 │                 │                     │    │
│  │  ┌──────▼───────┐  ┌──────▼───────┐  ┌──────▼───────┐            │    │
│  │  │  Clipboard   │  │   Hotkey     │  │  Persistence │            │    │
│  │  │   Engine     │  │   Engine     │  │    Engine    │            │    │
│  │  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘            │    │
│  │         │                 │                 │                     │    │
│  └─────────┼─────────────────┼─────────────────┼─────────────────────┘    │
│            │                 │                 │                          │
│  ┌─────────┼─────────────────┼─────────────────┼─────────────────────┐    │
│  │         │      PLATFORM ABSTRACTION LAYER   │                     │    │
│  │  ┌──────▼───────┐  ┌──────▼───────┐  ┌──────▼───────┐            │    │
│  │  │  Clipboard   │  │   Hotkey     │  │  Filesystem  │            │    │
│  │  │    Trait     │  │    Trait     │  │    Trait     │            │    │
│  │  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘            │    │
│  │         │                 │                 │                     │    │
│  │    ┌────┴────┐       ┌────┴────┐       ┌────┴────┐               │    │
│  │    │Win│Mac│Lin      │Win│Mac│Lin      │Win│Mac│Lin              │    │
│  │    └─────────┘       └─────────┘       └─────────┘               │    │
│  │                                                                   │    │
│  └───────────────────────────────────────────────────────────────────┘    │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         OPERATING SYSTEM                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                   │
│  │   System     │  │   Input      │  │    File      │                   │
│  │  Clipboard   │  │   Events     │  │   System     │                   │
│  └──────────────┘  └──────────────┘  └──────────────┘                   │
└─────────────────────────────────────────────────────────────────────────┘
```

### 4.2 Crate Structure

```
pockets/
├── Cargo.toml                 # Workspace root
├── crates/
│   ├── pockets-core/          # Platform-agnostic business logic
│   │   ├── src/
│   │   │   ├── lib.rs
│   │   │   ├── slot.rs        # Slot data structures
│   │   │   ├── manager.rs     # Slot manager
│   │   │   ├── event.rs       # Event definitions
│   │   │   └── config.rs      # Configuration
│   │   └── Cargo.toml
│   │
│   ├── pockets-clipboard/     # Clipboard abstraction
│   │   ├── src/
│   │   │   ├── lib.rs
│   │   │   ├── traits.rs      # ClipboardProvider trait
│   │   │   ├── content.rs     # ClipboardContent enum
│   │   │   ├── windows.rs     # Windows implementation
│   │   │   ├── macos.rs       # macOS implementation
│   │   │   └── linux.rs       # Linux (X11/Wayland)
│   │   └── Cargo.toml
│   │
│   ├── pockets-hotkey/        # Hotkey abstraction
│   │   ├── src/
│   │   │   ├── lib.rs
│   │   │   ├── traits.rs      # HotkeyProvider trait
│   │   │   ├── gesture.rs     # Tap/Hold detection
│   │   │   ├── windows.rs
│   │   │   ├── macos.rs
│   │   │   └── linux.rs
│   │   └── Cargo.toml
│   │
│   ├── pockets-persist/       # Storage abstraction
│   │   ├── src/
│   │   │   ├── lib.rs
│   │   │   ├── traits.rs
│   │   │   ├── sqlite.rs      # SQLite backend
│   │   │   └── recovery.rs    # Crash recovery
│   │   └── Cargo.toml
│   │
│   └── pockets-ui/            # Dioxus UI components
│       ├── src/
│       │   ├── lib.rs
│       │   ├── app.rs         # Main app component
│       │   ├── tray.rs        # System tray
│       │   ├── picker.rs      # Near-cursor slot picker
│       │   ├── preview.rs     # Content preview
│       │   └── theme.rs       # Styling
│       └── Cargo.toml
│
├── src/
│   └── main.rs                # Application entry point
│
├── assets/
│   ├── icons/                 # Slot icons (1-9, states)
│   └── fonts/                 # Embedded fonts if needed
│
├── docs/
│   ├── ARCHITECTURE.md        # This document
│   ├── PAST-PRESENT-FUTURE.md
│   └── platform/
│       ├── windows.md
│       ├── macos.md
│       └── linux.md
│
└── tests/
    ├── integration/
    └── e2e/
```

### 4.3 Dependency Graph (Simplified)

```
                    ┌─────────────┐
                    │   main.rs   │
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │ pockets-ui  │
                    └──────┬──────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
       ┌──────▼──────┐     │     ┌──────▼──────┐
       │   dioxus    │     │     │  tray-icon  │
       └─────────────┘     │     └─────────────┘
                           │
                    ┌──────▼──────┐
                    │pockets-core │
                    └──────┬──────┘
                           │
         ┌─────────────────┼─────────────────┐
         │                 │                 │
  ┌──────▼──────┐   ┌──────▼──────┐   ┌──────▼──────┐
  │  pockets-   │   │  pockets-   │   │  pockets-   │
  │  clipboard  │   │   hotkey    │   │   persist   │
  └──────┬──────┘   └──────┬──────┘   └──────┬──────┘
         │                 │                 │
  ┌──────▼──────┐   ┌──────▼──────┐   ┌──────▼──────┐
  │   arboard   │   │global-hotkey│   │   rusqlite  │
  │clipboard-rs │   │    rdev     │   │             │
  └─────────────┘   └─────────────┘   └─────────────┘
```

---

## 5. Platform Abstraction Layer

### 5.1 Design Principle

Every platform-specific operation is behind a trait. The core layer never imports platform modules directly.

```rust
// pockets-clipboard/src/traits.rs

/// Content that can be stored in a clipboard slot
#[derive(Clone, Debug)]
pub enum ClipboardContent {
    Text(String),
    Html { html: String, plain: Option<String> },
    Image(ImageData),
    Files(Vec<PathBuf>),
    Custom { format: String, data: Vec<u8> },
}

#[derive(Clone, Debug)]
pub struct ImageData {
    pub width: u32,
    pub height: u32,
    pub pixels: Vec<u8>,  // RGBA
}

/// Platform clipboard operations
#[async_trait]
pub trait ClipboardProvider: Send + Sync {
    /// Read current clipboard contents
    async fn read(&self) -> Result<ClipboardContent, ClipboardError>;
    
    /// Write content to clipboard
    async fn write(&self, content: &ClipboardContent) -> Result<(), ClipboardError>;
    
    /// Subscribe to clipboard changes
    fn subscribe(&self) -> broadcast::Receiver<ClipboardEvent>;
    
    /// Get available formats without reading content
    async fn available_formats(&self) -> Result<Vec<String>, ClipboardError>;
}

#[derive(Clone, Debug)]
pub struct ClipboardEvent {
    pub timestamp: Instant,
    pub source_app: Option<String>,  // Best-effort attribution
}

#[derive(Debug, thiserror::Error)]
pub enum ClipboardError {
    #[error("Clipboard is locked by another process")]
    Locked,
    #[error("Clipboard is empty")]
    Empty,
    #[error("Format not supported: {0}")]
    UnsupportedFormat(String),
    #[error("Platform error: {0}")]
    Platform(String),
}
```

### 5.2 Hotkey Abstraction

```rust
// pockets-hotkey/src/traits.rs

/// A keyboard gesture (tap vs hold)
#[derive(Clone, Debug, PartialEq)]
pub enum Gesture {
    Tap,
    Hold { duration: Duration },
}

/// Hotkey events we care about
#[derive(Clone, Debug)]
pub enum HotkeyEvent {
    Copy(Gesture),
    Paste(Gesture),
    SlotSelect { slot: u8 },  // 1-9
    Escape,
}

/// Platform hotkey operations
#[async_trait]
pub trait HotkeyProvider: Send + Sync {
    /// Start listening for hotkeys
    async fn start(&self) -> Result<(), HotkeyError>;
    
    /// Stop listening
    async fn stop(&self) -> Result<(), HotkeyError>;
    
    /// Subscribe to hotkey events
    fn subscribe(&self) -> broadcast::Receiver<HotkeyEvent>;
    
    /// Check if we have necessary permissions
    fn has_permissions(&self) -> bool;
    
    /// Request permissions (may show system dialog)
    async fn request_permissions(&self) -> Result<bool, HotkeyError>;
}
```

### 5.3 Platform Selection

```rust
// pockets-clipboard/src/lib.rs

pub fn create_clipboard_provider() -> Box<dyn ClipboardProvider> {
    #[cfg(target_os = "windows")]
    { Box::new(windows::WindowsClipboard::new()) }
    
    #[cfg(target_os = "macos")]
    { Box::new(macos::MacOSClipboard::new()) }
    
    #[cfg(target_os = "linux")]
    { 
        if std::env::var("WAYLAND_DISPLAY").is_ok() {
            Box::new(linux::WaylandClipboard::new())
        } else {
            Box::new(linux::X11Clipboard::new())
        }
    }
}
```

---

## 6. Clipboard Engine

### 6.1 Responsibilities

1. **Monitor** system clipboard for changes
2. **Capture** content when copy occurs
3. **Store** content in appropriate slot
4. **Serve** content to system clipboard when slot selected
5. **Track** active slot

### 6.2 Core Data Structures

```rust
// pockets-core/src/slot.rs

use chrono::{DateTime, Utc};
use uuid::Uuid;

/// A single clipboard slot
#[derive(Clone, Debug)]
pub struct Slot {
    pub id: u8,                           // 1-9
    pub content: Option<SlotContent>,
    pub state: SlotState,
}

#[derive(Clone, Debug)]
pub struct SlotContent {
    pub uuid: Uuid,                       // Unique ID for this capture
    pub data: ClipboardContent,           // The actual content
    pub captured_at: DateTime<Utc>,       // When captured
    pub source_app: Option<String>,       // Best-effort source attribution
    pub hash: u64,                         // For deduplication
}

#[derive(Clone, Copy, Debug, PartialEq)]
pub enum SlotState {
    Empty,      // Gray in UI
    Occupied,   // White in UI
    Active,     // Green in UI (and Occupied)
}
```

### 6.3 Slot Manager

```rust
// pockets-core/src/manager.rs

pub struct SlotManager {
    slots: [Slot; 9],
    active_slot: u8,
    clipboard: Arc<dyn ClipboardProvider>,
    persistence: Arc<dyn PersistenceProvider>,
    event_tx: broadcast::Sender<SlotEvent>,
}

#[derive(Clone, Debug)]
pub enum SlotEvent {
    ContentCaptured { slot: u8, content_id: Uuid },
    SlotSelected { slot: u8 },
    SlotCleared { slot: u8 },
    ActiveChanged { from: u8, to: u8 },
}

impl SlotManager {
    /// Called when clipboard change detected
    pub async fn on_clipboard_change(&mut self) -> Result<(), Error> {
        let content = self.clipboard.read().await?;
        let hash = self.compute_hash(&content);
        
        // Deduplication: don't re-capture if we just wrote this
        if self.is_echo(hash) {
            return Ok(());
        }
        
        // Capture to active slot
        let slot_content = SlotContent {
            uuid: Uuid::new_v4(),
            data: content,
            captured_at: Utc::now(),
            source_app: self.get_foreground_app(),
            hash,
        };
        
        self.slots[self.active_slot as usize - 1].content = Some(slot_content.clone());
        self.slots[self.active_slot as usize - 1].state = SlotState::Active;
        
        // Persist
        self.persistence.save_slot(self.active_slot, &slot_content).await?;
        
        // Notify
        self.event_tx.send(SlotEvent::ContentCaptured {
            slot: self.active_slot,
            content_id: slot_content.uuid,
        })?;
        
        Ok(())
    }
    
    /// Select a slot (for copy-to or paste-from)
    pub async fn select_slot(&mut self, slot: u8) -> Result<(), Error> {
        if slot < 1 || slot > 9 {
            return Err(Error::InvalidSlot(slot));
        }
        
        let old_active = self.active_slot;
        self.active_slot = slot;
        
        // Update states
        self.slots[old_active as usize - 1].state = 
            if self.slots[old_active as usize - 1].content.is_some() {
                SlotState::Occupied
            } else {
                SlotState::Empty
            };
        
        self.slots[slot as usize - 1].state = SlotState::Active;
        
        // If slot has content, write to system clipboard
        if let Some(content) = &self.slots[slot as usize - 1].content {
            self.clipboard.write(&content.data).await?;
        }
        
        self.event_tx.send(SlotEvent::ActiveChanged {
            from: old_active,
            to: slot,
        })?;
        
        Ok(())
    }
    
    /// Copy selection to specific slot
    pub async fn copy_to_slot(&mut self, slot: u8) -> Result<(), Error> {
        self.select_slot(slot).await?;
        // Content will be captured in on_clipboard_change
        Ok(())
    }
    
    /// Paste from specific slot
    pub async fn paste_from_slot(&mut self, slot: u8) -> Result<(), Error> {
        self.select_slot(slot).await?;
        // System clipboard now has slot content, normal paste will work
        Ok(())
    }
}
```

### 6.4 Echo Detection

When we write to the clipboard, we'll receive a "clipboard changed" event. We must not re-capture our own writes.

```rust
impl SlotManager {
    fn is_echo(&self, hash: u64) -> bool {
        // Check if this hash matches what we just wrote
        if let Some(last_write) = &self.last_write {
            if last_write.hash == hash 
               && last_write.timestamp.elapsed() < Duration::from_millis(500) {
                return true;
            }
        }
        false
    }
}
```

---

## 7. Hotkey Interception System

### 7.1 The Timing Challenge

We need to distinguish:
- **Tap** (<300ms): Ctrl pressed, C pressed, both released quickly
- **Hold** (≥1.5s): Ctrl pressed, C pressed, held for 1.5+ seconds

This requires intercepting key **down** and **up** events, not just "hotkey triggered."

### 7.2 Gesture Detection State Machine

```rust
// pockets-hotkey/src/gesture.rs

pub struct GestureDetector {
    state: GestureState,
    config: GestureConfig,
    event_tx: broadcast::Sender<HotkeyEvent>,
}

#[derive(Clone, Copy, Debug)]
pub struct GestureConfig {
    pub tap_threshold: Duration,   // 300ms default
    pub hold_threshold: Duration,  // 1500ms default
}

enum GestureState {
    Idle,
    CtrlDown { since: Instant },
    CtrlC { since: Instant, ctrl_down_at: Instant },
    CtrlV { since: Instant, ctrl_down_at: Instant },
    HoldTriggered { action: HoldAction },
}

enum HoldAction {
    Copy,
    Paste,
}

impl GestureDetector {
    pub fn on_key_event(&mut self, event: RawKeyEvent) {
        match (&self.state, event) {
            // Ctrl pressed
            (GestureState::Idle, RawKeyEvent::KeyDown(Key::Ctrl)) => {
                self.state = GestureState::CtrlDown { since: Instant::now() };
            }
            
            // C pressed while Ctrl held
            (GestureState::CtrlDown { since: ctrl_down }, RawKeyEvent::KeyDown(Key::C)) => {
                self.state = GestureState::CtrlC { 
                    since: Instant::now(),
                    ctrl_down_at: *ctrl_down,
                };
                self.start_hold_timer(HoldAction::Copy);
            }
            
            // V pressed while Ctrl held
            (GestureState::CtrlDown { since: ctrl_down }, RawKeyEvent::KeyDown(Key::V)) => {
                self.state = GestureState::CtrlV { 
                    since: Instant::now(),
                    ctrl_down_at: *ctrl_down,
                };
                self.start_hold_timer(HoldAction::Paste);
            }
            
            // C released while in CtrlC state (TAP detected)
            (GestureState::CtrlC { since, .. }, RawKeyEvent::KeyUp(Key::C)) => {
                let duration = since.elapsed();
                if duration < self.config.tap_threshold {
                    self.event_tx.send(HotkeyEvent::Copy(Gesture::Tap)).ok();
                }
                self.state = GestureState::CtrlDown { since: Instant::now() };
            }
            
            // Hold timer fired
            (GestureState::CtrlC { since, .. }, RawKeyEvent::HoldTimerFired) 
                if since.elapsed() >= self.config.hold_threshold => {
                self.event_tx.send(HotkeyEvent::Copy(Gesture::Hold { 
                    duration: since.elapsed() 
                })).ok();
                self.state = GestureState::HoldTriggered { action: HoldAction::Copy };
            }
            
            // Number key while hold triggered
            (GestureState::HoldTriggered { .. }, RawKeyEvent::KeyDown(Key::Num(n))) 
                if (1..=9).contains(&n) => {
                self.event_tx.send(HotkeyEvent::SlotSelect { slot: n }).ok();
            }
            
            // Ctrl released - reset
            (_, RawKeyEvent::KeyUp(Key::Ctrl)) => {
                self.state = GestureState::Idle;
            }
            
            // Escape - cancel
            (_, RawKeyEvent::KeyDown(Key::Escape)) => {
                self.event_tx.send(HotkeyEvent::Escape).ok();
                self.state = GestureState::Idle;
            }
            
            _ => {}
        }
    }
}
```

### 7.3 Platform Implementation Notes

**Windows:** Use `SetWindowsHookEx` with `WH_KEYBOARD_LL` for low-level keyboard hook. This captures keys before any application sees them.

**macOS:** Use `CGEventTap` with `kCGEventKeyDown` and `kCGEventKeyUp`. Requires Accessibility permission.

**Linux X11:** Use `XGrabKey` or monitor via `XRecord` extension.

**Linux Wayland:** This is problematic. Wayland deliberately prevents global hotkey capture for security. Options:
- Use `libinput` with elevated permissions
- Require user to run as root (bad)
- Use portal API if compositor supports it
- Degrade gracefully: tray-only interaction, no hotkeys

---

## 8. UI Layer

### 8.1 Dioxus Component Architecture

```rust
// pockets-ui/src/app.rs

use dioxus::prelude::*;

pub fn App(cx: Scope) -> Element {
    // Global state
    let slots = use_shared_state::<SlotState>(cx);
    let ui_state = use_shared_state::<UIState>(cx);
    
    cx.render(rsx! {
        // System tray (always present)
        SystemTray {
            slots: slots,
            on_slot_click: |slot| { /* ... */ }
        }
        
        // Near-cursor picker (conditionally rendered)
        if ui_state.picker_visible {
            rsx! {
                SlotPicker {
                    position: ui_state.cursor_position,
                    slots: slots,
                    mode: ui_state.picker_mode,
                    on_select: |slot| { /* ... */ },
                    on_dismiss: || { /* ... */ }
                }
            }
        }
        
        // Preview popup (on hover)
        if let Some(preview) = &ui_state.preview {
            rsx! {
                ContentPreview {
                    content: preview.content,
                    position: preview.position,
                }
            }
        }
    })
}
```

### 8.2 Slot Picker Component

```rust
// pockets-ui/src/picker.rs

#[derive(Props)]
pub struct SlotPickerProps<'a> {
    position: CursorPosition,
    slots: &'a [Slot; 9],
    mode: PickerMode,
    on_select: EventHandler<'a, u8>,
    on_dismiss: EventHandler<'a, ()>,
}

pub fn SlotPicker<'a>(cx: Scope<'a, SlotPickerProps<'a>>) -> Element {
    let visible_slots = use_state(cx, || cx.props.slots.iter()
        .filter(|s| s.state != SlotState::Empty || s.id <= 2)  // Always show 1-2
        .count());
    
    let fade_progress = use_animation(cx, /* fade logic */);
    
    cx.render(rsx! {
        div {
            class: "slot-picker",
            style: "
                position: fixed;
                left: {cx.props.position.x + 20}px;
                top: {cx.props.position.y}px;
                display: flex;
                gap: 4px;
                opacity: {fade_progress};
                pointer-events: auto;
            ",
            
            for slot in cx.props.slots.iter().take(*visible_slots.get()) {
                SlotIcon {
                    key: "{slot.id}",
                    slot: slot,
                    on_click: move |_| cx.props.on_select.call(slot.id),
                    on_hover: move |_| { /* show preview */ },
                }
            }
        }
    })
}
```

### 8.3 Visual Design Specifications

```
SLOT ICON STATES
================

┌─────────────────────────────────────────────────────────┐
│                                                          │
│   ACTIVE (Green)        OCCUPIED (White)    EMPTY (Gray) │
│   ┌──────────┐          ┌──────────┐        ┌──────────┐│
│   │ ┌──────┐ │          │ ┌──────┐ │        │ ┌──────┐ ││
│   │ │  1   │ │          │ │  2   │ │        │ │  3   │ ││
│   │ │ ──── │ │          │ │ ──── │ │        │ │      │ ││
│   │ │ ──── │ │          │ │ ──── │ │        │ │      │ ││
│   │ └──────┘ │          │ └──────┘ │        │ └──────┘ ││
│   └──────────┘          └──────────┘        └──────────┘│
│   #22C55E               #FFFFFF             #6B7280     │
│   (green-500)           (white)             (gray-500)  │
│                                                          │
└─────────────────────────────────────────────────────────┘

Icon Size: 24x24px (normal), 32x32px (hover)
Icon Spacing: 4px gap
Drop Shadow: 0 2px 8px rgba(0,0,0,0.15)
Border Radius: 4px
Transition: 150ms ease-out (all properties)

FADE ANIMATION
==============
0ms    → opacity: 0, scale: 0.95
50ms   → opacity: 1, scale: 1.0
1700ms → opacity: 1 (hold)
2000ms → opacity: 0, scale: 0.95 (fade out)

HOVER BEHAVIOR
==============
- Hover pauses fade timer
- Hover scales icon to 32x32
- After 200ms hover, show preview tooltip
- Preview appears above icon, max 150x150px
```

### 8.4 System Tray Integration

```rust
// pockets-ui/src/tray.rs

use tray_icon::{TrayIcon, TrayIconBuilder};
use muda::{Menu, MenuItem};

pub struct PocketsTray {
    icon: TrayIcon,
    menu: Menu,
    slots: Arc<RwLock<[Slot; 9]>>,
}

impl PocketsTray {
    pub fn new(slots: Arc<RwLock<[Slot; 9]>>) -> Result<Self, Error> {
        let menu = Menu::new();
        
        // Dynamic slot items
        for i in 1..=9 {
            let item = MenuItem::new(format!("Slot {}", i), true, None);
            menu.append(&item)?;
        }
        
        menu.append(&MenuItem::separator())?;
        menu.append(&MenuItem::new("Settings...", true, None))?;
        menu.append(&MenuItem::new("Quit", true, None))?;
        
        let icon = TrayIconBuilder::new()
            .with_menu(Box::new(menu.clone()))
            .with_icon(Self::generate_icon(&slots.read().unwrap()))
            .with_tooltip("Pockets - Clipboard Manager")
            .build()?;
        
        Ok(Self { icon, menu, slots })
    }
    
    /// Regenerate tray icon to reflect current state
    fn generate_icon(slots: &[Slot; 9]) -> Icon {
        // Dynamically generate icon showing:
        // - Number of active slot
        // - Visual indicator of occupied slots
        // Implementation depends on platform icon capabilities
    }
}
```

---

## 9. Storage & Persistence

### 9.1 Requirements

- **Survive crashes:** Content must not be lost if app crashes
- **Survive reboots:** Content persists across system restarts
- **Fast startup:** App should be usable within 100ms of launch
- **Efficient:** Don't store duplicate content
- **Bounded:** Configurable max storage size

### 9.2 SQLite Schema

```sql
-- schema.sql

-- Metadata table for versioning
CREATE TABLE IF NOT EXISTS meta (
    key TEXT PRIMARY KEY,
    value TEXT NOT NULL
);

INSERT OR IGNORE INTO meta (key, value) VALUES ('schema_version', '1');

-- Content storage (deduplicated)
CREATE TABLE IF NOT EXISTS content (
    hash INTEGER PRIMARY KEY,           -- xxHash64 of content
    content_type TEXT NOT NULL,         -- 'text', 'html', 'image', 'files', 'custom'
    data BLOB NOT NULL,                 -- Serialized content
    size_bytes INTEGER NOT NULL,
    created_at TEXT NOT NULL,           -- ISO8601
    last_accessed_at TEXT NOT NULL,
    access_count INTEGER DEFAULT 1
);

-- Slot state
CREATE TABLE IF NOT EXISTS slots (
    slot_id INTEGER PRIMARY KEY,        -- 1-9
    content_hash INTEGER,               -- FK to content.hash (nullable)
    captured_at TEXT,                   -- ISO8601
    source_app TEXT,
    FOREIGN KEY (content_hash) REFERENCES content(hash)
);

-- Initialize slots
INSERT OR IGNORE INTO slots (slot_id) VALUES (1), (2), (3), (4), (5), (6), (7), (8), (9);

-- Active slot tracking
CREATE TABLE IF NOT EXISTS state (
    key TEXT PRIMARY KEY,
    value TEXT NOT NULL
);

INSERT OR IGNORE INTO state (key, value) VALUES ('active_slot', '1');

-- Write-ahead log for crash recovery
CREATE TABLE IF NOT EXISTS wal (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    operation TEXT NOT NULL,            -- 'slot_update', 'content_add'
    data TEXT NOT NULL,                 -- JSON payload
    created_at TEXT NOT NULL,
    applied INTEGER DEFAULT 0
);

-- Indices
CREATE INDEX IF NOT EXISTS idx_content_last_accessed ON content(last_accessed_at);
CREATE INDEX IF NOT EXISTS idx_wal_applied ON wal(applied);
```

### 9.3 Persistence Provider

```rust
// pockets-persist/src/sqlite.rs

pub struct SqlitePersistence {
    pool: Pool<Sqlite>,
    max_storage_bytes: u64,
}

impl SqlitePersistence {
    pub async fn new(path: &Path) -> Result<Self, Error> {
        let pool = SqlitePoolOptions::new()
            .max_connections(1)  // SQLite is single-writer
            .connect(&format!("sqlite:{}?mode=rwc", path.display()))
            .await?;
        
        // Run migrations
        sqlx::migrate!("./migrations").run(&pool).await?;
        
        // Recover from any incomplete operations
        Self::recover(&pool).await?;
        
        Ok(Self { 
            pool, 
            max_storage_bytes: 100 * 1024 * 1024,  // 100MB default
        })
    }
    
    async fn recover(pool: &Pool<Sqlite>) -> Result<(), Error> {
        // Replay any unapplied WAL entries
        let pending: Vec<WalEntry> = sqlx::query_as(
            "SELECT * FROM wal WHERE applied = 0 ORDER BY id"
        ).fetch_all(pool).await?;
        
        for entry in pending {
            // Replay operation
            Self::apply_wal_entry(pool, &entry).await?;
        }
        
        // Clear applied entries
        sqlx::query("DELETE FROM wal WHERE applied = 1")
            .execute(pool)
            .await?;
        
        Ok(())
    }
    
    pub async fn save_slot(&self, slot: u8, content: &SlotContent) -> Result<(), Error> {
        let mut tx = self.pool.begin().await?;
        
        // Write to WAL first
        let wal_data = serde_json::to_string(&WalPayload::SlotUpdate {
            slot,
            content: content.clone(),
        })?;
        
        sqlx::query("INSERT INTO wal (operation, data, created_at) VALUES (?, ?, ?)")
            .bind("slot_update")
            .bind(&wal_data)
            .bind(Utc::now().to_rfc3339())
            .execute(&mut *tx)
            .await?;
        
        // Insert content (deduplicated by hash)
        let serialized = bincode::serialize(&content.data)?;
        sqlx::query(
            "INSERT OR IGNORE INTO content (hash, content_type, data, size_bytes, created_at, last_accessed_at)
             VALUES (?, ?, ?, ?, ?, ?)"
        )
        .bind(content.hash as i64)
        .bind(content.data.type_name())
        .bind(&serialized)
        .bind(serialized.len() as i64)
        .bind(content.captured_at.to_rfc3339())
        .bind(content.captured_at.to_rfc3339())
        .execute(&mut *tx)
        .await?;
        
        // Update slot
        sqlx::query(
            "UPDATE slots SET content_hash = ?, captured_at = ?, source_app = ? WHERE slot_id = ?"
        )
        .bind(content.hash as i64)
        .bind(content.captured_at.to_rfc3339())
        .bind(&content.source_app)
        .bind(slot)
        .execute(&mut *tx)
        .await?;
        
        // Mark WAL entry as applied
        sqlx::query("UPDATE wal SET applied = 1 WHERE id = last_insert_rowid()")
            .execute(&mut *tx)
            .await?;
        
        tx.commit().await?;
        
        // Enforce storage limits asynchronously
        self.enforce_storage_limits().await;
        
        Ok(())
    }
}
```

### 9.4 Storage Location

| Platform | Location |
|----------|----------|
| Windows | `%APPDATA%\Pockets\` |
| macOS | `~/Library/Application Support/Pockets/` |
| Linux | `~/.local/share/pockets/` |

Files:
- `pockets.db` - SQLite database
- `pockets.db-wal` - SQLite WAL file
- `pockets.db-shm` - SQLite shared memory

---

## 10. Security Model

### 10.1 Threat Model

| Threat | Mitigation |
|--------|------------|
| Sensitive data in slots (passwords, SSNs) | Honor `ConcealedType` markers from password managers; configurable auto-clear timeout |
| Malicious app reading our storage | OS file permissions; optional encryption at rest |
| Clipboard history as attack vector | Bounded history size; no cloud sync; user controls retention |
| Keystroke logging accusations | We don't log keystrokes; we detect gestures only |
| Privilege escalation | Run as normal user; no elevated permissions except where required (Wayland) |

### 10.2 Sensitive Data Detection

```rust
// pockets-core/src/security.rs

pub struct SensitivityDetector {
    patterns: Vec<SensitivePattern>,
}

struct SensitivePattern {
    name: &'static str,
    regex: Regex,
}

impl SensitivityDetector {
    pub fn new() -> Self {
        Self {
            patterns: vec![
                SensitivePattern {
                    name: "SSN",
                    regex: Regex::new(r"\b\d{3}-\d{2}-\d{4}\b").unwrap(),
                },
                SensitivePattern {
                    name: "Credit Card",
                    regex: Regex::new(r"\b\d{4}[- ]?\d{4}[- ]?\d{4}[- ]?\d{4}\b").unwrap(),
                },
                SensitivePattern {
                    name: "API Key",
                    regex: Regex::new(r"\b(sk|pk|api)[_-]?(live|test)?[_-]?[a-zA-Z0-9]{20,}\b").unwrap(),
                },
            ],
        }
    }
    
    pub fn is_sensitive(&self, content: &ClipboardContent) -> Option<&'static str> {
        if let ClipboardContent::Text(text) = content {
            for pattern in &self.patterns {
                if pattern.regex.is_match(text) {
                    return Some(pattern.name);
                }
            }
        }
        None
    }
}
```

### 10.3 Password Manager Integration

Many password managers mark clipboard entries as "concealed" or "transient":
- macOS: `org.nspasteboard.ConcealedType`
- macOS: `org.nspasteboard.TransientType`
- Windows: Custom clipboard format markers
- Android: `ClipDescription.EXTRA_IS_SENSITIVE`

```rust
impl SlotManager {
    async fn should_capture(&self, content: &ClipboardContent, metadata: &ClipboardMetadata) -> bool {
        // Honor concealed/transient markers
        if metadata.is_concealed || metadata.is_transient {
            return false;
        }
        
        // User-configured app exclusions
        if let Some(app) = &metadata.source_app {
            if self.config.excluded_apps.contains(app) {
                return false;
            }
        }
        
        true
    }
}
```

---

## 11. Performance Requirements

### 11.1 Latency Budgets

| Operation | Budget | Notes |
|-----------|--------|-------|
| Tap Ctrl+C → content captured | <10ms | Must feel instant |
| Tap Ctrl+V → paste occurs | <10ms | Must feel instant |
| Icon appearance after tap | <50ms | Perceptible but acceptable |
| Hold detection → picker visible | <100ms | After threshold reached |
| Slot selection → clipboard write | <20ms | Must feel instant |
| App cold start → ready | <500ms | Before user notices |
| App warm start (from tray) | <100ms | |

### 11.2 Memory Budgets

| Component | Budget | Notes |
|-----------|--------|-------|
| Base process | <20MB | Idle, no content |
| Per text slot | <1MB | Typical text |
| Per image slot | <10MB | Uncompressed in memory |
| Max total | <100MB | All slots full |

### 11.3 Storage Budgets

| Resource | Budget | Notes |
|----------|--------|-------|
| SQLite DB | <100MB | Default, configurable |
| Per-slot max | <50MB | Prevent single giant capture |

### 11.4 CPU Budgets

| State | Budget | Notes |
|-------|--------|-------|
| Idle | <0.1% | Just listening |
| Copy/Paste | <1% | Brief spike |
| UI visible | <2% | Rendering |

---

## 12. Platform-Specific Considerations

### 12.1 Windows

**Clipboard API:**
- Use `AddClipboardFormatListener()` for change notifications
- Use `OpenClipboard()` / `GetClipboardData()` / `SetClipboardData()` / `CloseClipboard()` for access
- Must handle `WM_CLIPBOARDUPDATE` messages

**Hotkeys:**
- `SetWindowsHookEx` with `WH_KEYBOARD_LL` for low-level hook
- Runs in hook thread, must be fast
- Works globally without special permissions

**Gotchas:**
- Clipboard can be locked by other apps; retry with backoff
- Some apps use delayed rendering; handle `WM_RENDERFORMAT`
- UAC elevation changes hook behavior

### 12.2 macOS

**Clipboard API:**
- `NSPasteboard.general` for system clipboard
- Poll `changeCount` for change detection (no notification API)
- 100ms polling interval is reasonable

**Hotkeys:**
- `CGEventTap` for global keyboard monitoring
- **Requires Accessibility permission**
- Must prompt user and handle denial gracefully

**Gotchas:**
- Sandboxed apps have clipboard restrictions
- **Cannot distribute clipboard managers via Mac App Store** (sandbox blocks required APIs)
- Must distribute via direct download with notarization

### 12.3 Linux (X11)

**Clipboard API:**
- Use `XFixes` extension for clipboard change notifications
- `XConvertSelection` for reading content
- `XSetSelectionOwner` for writing

**Hotkeys:**
- `XGrabKey` for global hotkeys
- Or `XRecord` extension for monitoring

**Gotchas:**
- Clipboard is owned by the copying app; content disappears if app closes
- Must "re-own" clipboard data to persist it
- Multiple "selections": PRIMARY (middle-click), CLIPBOARD (Ctrl+C)

### 12.3 Linux (Wayland)

**Clipboard API:**
- `wl-clipboard` or `wlr-data-control` protocol
- Compositor must support `data-control` extension
- More secure than X11 (only focused app can read by default)

**Hotkeys:**
- **Global hotkeys are deliberately unsupported** in Wayland
- Options:
  1. Use `libinput` with elevated permissions (requires root or input group)
  2. Use compositor-specific protocol if available
  3. **Graceful degradation:** Tray-only interaction, no global hotkeys

**Recommended Wayland Strategy:**
```
IF compositor supports global-shortcuts portal:
    Use portal API
ELSE IF user is in 'input' group:
    Use libinput with rdev
ELSE:
    Warn user; offer tray-only mode
```

---

## 13. Error Handling Philosophy

### 13.1 The Three Categories

**1. Expected Errors (Handle Gracefully)**
- Clipboard locked by another app → Retry with backoff
- Clipboard empty → No-op
- Network unavailable → N/A (we don't use network)

**2. Unexpected Errors (Log and Recover)**
- Database write failed → Retry, fall back to in-memory
- Permission denied → Prompt user, degrade gracefully

**3. Unrecoverable Errors (Crash Loudly)**
- Corrupted database → Delete and recreate
- Out of memory → Panic with clear message

### 13.2 Error Type Design

```rust
// pockets-core/src/error.rs

#[derive(Debug, thiserror::Error)]
pub enum PocketsError {
    // Expected - handle gracefully
    #[error("Clipboard is temporarily locked")]
    ClipboardLocked,
    
    #[error("Clipboard is empty")]
    ClipboardEmpty,
    
    #[error("Slot {0} is empty")]
    SlotEmpty(u8),
    
    // Unexpected - log and recover
    #[error("Persistence error: {0}")]
    Persistence(#[from] PersistenceError),
    
    #[error("Platform error: {0}")]
    Platform(String),
    
    // Permission issues - prompt user
    #[error("Accessibility permission required")]
    AccessibilityPermissionDenied,
    
    #[error("Input group membership required for Wayland hotkeys")]
    InputGroupRequired,
}
```

### 13.3 No Panics Policy

We use `.unwrap()` and `.expect()` **only** in:
1. Tests
2. Static initialization that cannot fail
3. Invariants that indicate bugs (with clear messages)

All other error paths must be handled explicitly.

---

## 14. Testing Strategy

### 14.1 Unit Tests

Every module has unit tests. Core logic is platform-agnostic and easily testable.

```rust
#[cfg(test)]
mod tests {
    use super::*;
    
    #[test]
    fn gesture_detector_tap() {
        let mut detector = GestureDetector::new(GestureConfig::default());
        let (tx, mut rx) = broadcast::channel(10);
        detector.event_tx = tx;
        
        // Simulate Ctrl+C tap
        detector.on_key_event(RawKeyEvent::KeyDown(Key::Ctrl));
        detector.on_key_event(RawKeyEvent::KeyDown(Key::C));
        std::thread::sleep(Duration::from_millis(100));
        detector.on_key_event(RawKeyEvent::KeyUp(Key::C));
        detector.on_key_event(RawKeyEvent::KeyUp(Key::Ctrl));
        
        let event = rx.try_recv().unwrap();
        assert!(matches!(event, HotkeyEvent::Copy(Gesture::Tap)));
    }
    
    #[test]
    fn gesture_detector_hold() {
        let mut detector = GestureDetector::new(GestureConfig {
            hold_threshold: Duration::from_millis(100),
            ..Default::default()
        });
        
        detector.on_key_event(RawKeyEvent::KeyDown(Key::Ctrl));
        detector.on_key_event(RawKeyEvent::KeyDown(Key::C));
        std::thread::sleep(Duration::from_millis(150));
        detector.on_key_event(RawKeyEvent::HoldTimerFired);
        
        // Should have received Hold event
    }
}
```

### 14.2 Integration Tests

Test platform abstraction implementations against real OS APIs.

```rust
// tests/integration/clipboard_test.rs

#[test]
#[ignore] // Requires real clipboard access
fn test_clipboard_roundtrip() {
    let clipboard = create_clipboard_provider();
    
    let content = ClipboardContent::Text("test content".to_string());
    block_on(clipboard.write(&content)).unwrap();
    
    let read = block_on(clipboard.read()).unwrap();
    assert_eq!(read, content);
}
```

### 14.3 End-to-End Tests

Automated UI tests using platform-specific tooling.

**Windows:** UI Automation API
**macOS:** XCUITest
**Linux:** AT-SPI or Dogtail

### 14.4 Fuzzing

Fuzz the gesture detector and clipboard content parsing:

```rust
#![no_main]
use libfuzzer_sys::fuzz_target;
use pockets_hotkey::gesture::GestureDetector;

fuzz_target!(|data: &[u8]| {
    let mut detector = GestureDetector::new(Default::default());
    for byte in data {
        let event = match byte % 6 {
            0 => RawKeyEvent::KeyDown(Key::Ctrl),
            1 => RawKeyEvent::KeyUp(Key::Ctrl),
            2 => RawKeyEvent::KeyDown(Key::C),
            3 => RawKeyEvent::KeyUp(Key::C),
            4 => RawKeyEvent::KeyDown(Key::V),
            5 => RawKeyEvent::KeyUp(Key::V),
            _ => unreachable!(),
        };
        detector.on_key_event(event);
    }
});
```

---

## 15. Distribution & Packaging

### 15.1 Distribution Strategy

| Platform | Method | Notes |
|----------|--------|-------|
| Windows | Direct download (MSI/EXE), winget | No Store (avoids review complexity) |
| macOS | Direct download (DMG), Homebrew | Cannot use App Store (sandbox) |
| Linux | Flatpak, AppImage, AUR | Wayland support varies |

### 15.2 Code Signing

**Windows:**
- EV Code Signing Certificate required to avoid SmartScreen warnings
- Sign both installer and executable

**macOS:**
- Developer ID certificate required
- Must notarize with Apple
- Staple notarization ticket to DMG

### 15.3 Auto-Update

Implement auto-update for direct downloads:

```rust
// Update check on startup (async, non-blocking)
pub async fn check_for_updates() -> Option<UpdateInfo> {
    let current = env!("CARGO_PKG_VERSION");
    let latest = reqwest::get("https://pockets.app/version.json")
        .await.ok()?
        .json::<VersionInfo>()
        .await.ok()?;
    
    if semver::Version::parse(&latest.version) > semver::Version::parse(current) {
        Some(UpdateInfo {
            version: latest.version,
            download_url: latest.download_url,
            release_notes: latest.release_notes,
        })
    } else {
        None
    }
}
```

### 15.4 Build Matrix

| Target | Triple | Notes |
|--------|--------|-------|
| Windows x64 | `x86_64-pc-windows-msvc` | Primary |
| Windows ARM | `aarch64-pc-windows-msvc` | Secondary |
| macOS Intel | `x86_64-apple-darwin` | Primary |
| macOS ARM | `aarch64-apple-darwin` | Primary |
| Linux x64 | `x86_64-unknown-linux-gnu` | Primary |
| Linux ARM | `aarch64-unknown-linux-gnu` | Secondary |

---

## 16. Future Considerations

### 16.1 Potential Features (Not In Scope for v1)

| Feature | Complexity | Value | Notes |
|---------|-----------|-------|-------|
| Search across slots | Medium | Medium | Full-text search |
| Slot pinning | Low | Low | Prevent accidental overwrite |
| Slot naming | Low | Low | Custom labels |
| Export/Import | Medium | Low | Backup/restore |
| Snippets | High | Medium | Permanent saved clips |
| Cloud sync | High | Medium | End-to-end encrypted |
| OCR from images | High | Medium | Text extraction |
| Mobile companion | Very High | Low | Phone integration |

### 16.2 Anti-Features (Never Implementing)

| Feature | Reason |
|---------|--------|
| Telemetry | Privacy principle |
| Ads | Product integrity |
| Required account | Friction principle |
| Subscription pricing | User preference (one-time $12) |

### 16.3 Success Metrics

| Metric | Target | Measurement |
|--------|--------|-------------|
| Time to first copy | <1 minute | From install to successful copy to slot 2 |
| Feature discovery | >50% | Users who use slot selection within first week |
| Crash rate | <0.1% | Crashes per session |
| Memory | <50MB | 95th percentile during normal use |
| Startup time | <500ms | Cold start to responsive |

---

## Appendix A: Crate Dependencies

```toml
# Workspace Cargo.toml
[workspace]
members = [
    "crates/pockets-core",
    "crates/pockets-clipboard", 
    "crates/pockets-hotkey",
    "crates/pockets-persist",
    "crates/pockets-ui",
]

[workspace.dependencies]
# Async runtime
tokio = { version = "1", features = ["full"] }

# Serialization
serde = { version = "1", features = ["derive"] }
serde_json = "1"
bincode = "1"

# Database
sqlx = { version = "0.7", features = ["runtime-tokio", "sqlite"] }

# Error handling
thiserror = "1"
anyhow = "1"

# Logging
tracing = "0.1"
tracing-subscriber = "0.3"

# Utilities
uuid = { version = "1", features = ["v4", "serde"] }
chrono = { version = "0.4", features = ["serde"] }
xxhash-rust = { version = "0.8", features = ["xxh64"] }

# Clipboard
arboard = "3"
clipboard-rs = "0.1"

# Hotkeys  
global-hotkey = "0.5"
rdev = "0.5"

# UI
dioxus = { version = "0.5", features = ["desktop"] }
tray-icon = "0.11"
muda = "0.11"

# Platform-specific
[target.'cfg(windows)'.workspace.dependencies]
windows = { version = "0.52", features = ["Win32_UI_Input_KeyboardAndMouse", "Win32_System_DataExchange"] }

[target.'cfg(target_os = "macos")'.workspace.dependencies]
objc2 = "0.4"
objc2-app-kit = "0.1"

[target.'cfg(target_os = "linux")'.workspace.dependencies]
x11rb = "0.13"
wayland-client = "0.31"
```

---

## Appendix B: Configuration File Format

```toml
# ~/.config/pockets/config.toml (Linux)
# ~/Library/Application Support/Pockets/config.toml (macOS)
# %APPDATA%\Pockets\config.toml (Windows)

[general]
# Number of slots (1-9)
slot_count = 5

# Start on system login
autostart = true

# Show tray icon
show_tray = true

[timing]
# Milliseconds
tap_threshold = 300
hold_threshold = 1500
icon_display = 2000
icon_fade = 300

[storage]
# Maximum database size in MB
max_size_mb = 100

# Auto-clear sensitive data after N seconds (0 = disabled)
sensitive_timeout_seconds = 30

[security]
# Apps to exclude from capture
excluded_apps = [
    "1Password",
    "Bitwarden",
    "KeePassXC",
]

# Encrypt database at rest
encrypt_storage = false

[appearance]
# Icon theme: "auto", "light", "dark"
theme = "auto"
```

---

## Appendix C: Glossary

| Term | Definition |
|------|------------|
| Active Slot | The currently selected slot that receives copies and provides pastes |
| Capture | The act of copying content from system clipboard to a slot |
| Echo | A clipboard change caused by our own write (must be ignored) |
| Gesture | A tap or hold interaction pattern |
| Picker | The near-cursor UI showing slot icons |
| Serve | Writing slot content to system clipboard for pasting |
| Slot | One of 9 storage locations for clipboard content |
| Stage | Metaphor for the system clipboard (what's visible to OS/apps) |

---

**Document End**

*This architecture document is a living specification. Update it as implementation reveals new constraints or opportunities.*
