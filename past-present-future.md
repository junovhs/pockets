# Past / Present / Future

**Status:** Canonical (living snapshot)  
**Last updated:** 2025-12-30  
**Canonical policy:** This document states the current operational reality and the single next action.

---

## 1) Past (How we got here)

### The Research Phase (Completed)

**Competitive Analysis:**
- Analyzed Ditto (Windows), Maccy (macOS), Paste (macOS), CopyQ (cross-platform)
- Key insight: subscription fatigue is real ($30/year Paste generates backlash)
- Key insight: existing tools either look dated (Ditto) or are platform-locked (Paste)
- Gap identified: premium quality, cross-platform, one-time purchase under $15

**UX Research:**
- Studied Larry Tesler's Object→Verb inversion (1974-1975)
- Key insight: Copy/paste hasn't been rethought in 40 years
- Key insight: The clipboard is the only invisible element in modern UIs
- Conclusion: We won't invent a new paradigm, but we'll execute beautifully

**Interaction Design:**
- Rejected "spreading clippings on desk" (just describes existing managers)
- Rejected "two hands" metaphor (just Ditto with fewer slots)
- Landed on: Progressive disclosure with near-cursor UI
- Principle: Ctrl+C and Ctrl+V are sacred—never override tap behavior

**Business Model:**
- Pricing: $12 one-time purchase (sweet spot from competitive analysis)
- Distribution: Direct download only (Mac App Store sandbox blocks required APIs)
- Open source: MIT license, paid binaries (power users build from source, become contributors)

**Tech Stack Selection:**
- Language: Rust (safety, performance, cross-platform)
- UI Framework: Dioxus (not Tauri—native feel, <5MB binary)
- Clipboard: arboard + clipboard-rs
- Hotkeys: global-hotkey + rdev
- Persistence: SQLite via sqlx
- System tray: tray-icon + muda

### Key Decisions Locked

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Name | Pockets | "Extra pockets for your clipboard" - physical metaphor, unpretentious |
| Slots | 9 | Keyboard numbers 1-9, more than anyone needs |
| Interaction | Hold-to-reveal | Progressive disclosure, zero learning curve |
| Active Slot | Always one | Single answer to "what will paste?" |
| Cloud Sync | Never | Privacy principle, simplicity |
| Subscription | Never | User preference, competitive differentiation |

---

## 2) Present (Where we are right now)

**Status:** ARCHITECTURE COMPLETE — Ready for implementation

The architecture is specified. No code exists yet. This is intentional—we designed before building.

### What Exists

| Artifact | Status | Location |
|----------|--------|----------|
| Technical Architecture | ✅ Complete | `docs/ARCHITECTURE.md` |
| Past/Present/Future | ✅ Complete | `docs/PAST-PRESENT-FUTURE.md` |
| README | ✅ Complete | `README.md` |
| Code | ❌ None | — |

### Architecture Summary

```
┌─────────────────────────────────────────────────────┐
│                    POCKETS                          │
│                                                     │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐ │
│  │  pockets-ui │  │pockets-core │  │pockets-persist│
│  │   (Dioxus)  │  │  (slots)    │  │  (SQLite)   │ │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘ │
│         │                │                │        │
│         └────────────────┼────────────────┘        │
│                          │                          │
│  ┌───────────────────────┼───────────────────────┐ │
│  │         PLATFORM ABSTRACTION LAYER            │ │
│  │  ┌─────────────┐           ┌─────────────┐    │ │
│  │  │  clipboard  │           │   hotkey    │    │ │
│  │  │   (trait)   │           │   (trait)   │    │ │
│  │  └─────────────┘           └─────────────┘    │ │
│  │       │                         │             │ │
│  │  ┌────┴────┐               ┌────┴────┐        │ │
│  │  │Win│Mac│Lin              │Win│Mac│Lin       │ │
│  │  └─────────┘               └─────────┘        │ │
│  └───────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────┘
```

### The Three Laws (Locked)

1. **Law of Sanctity:** Ctrl+C/V tap behavior is indistinguishable from native
2. **Law of Visibility:** Clipboard state is always knowable without opening windows
3. **Law of Resilience:** Data survives crashes, reboots, and updates

### Known Challenges

| Challenge | Platform | Mitigation |
|-----------|----------|------------|
| No global hotkeys | Wayland | Graceful degradation to tray-only mode |
| Sandbox blocks clipboard monitoring | macOS App Store | Direct distribution only |
| Clipboard can be locked | Windows | Retry with exponential backoff |
| No clipboard change notifications | macOS | Poll changeCount at 100ms intervals |

---

## 3) Future (What we do next)

### Immediate Next Action

**Scaffold the Dioxus project and implement clipboard monitoring on ONE platform.**

| Phase | Description | Outcome | Effort |
|-------|-------------|---------|--------|
| 0 | Project scaffold | `cargo build` succeeds | 1-2 hrs |
| 1 | Clipboard monitoring (Windows) | Captures copies to console | 2-3 hrs |
| 2 | System tray | Icon appears, menu works | 2-3 hrs |
| 3 | Slot storage (in-memory) | 9 slots, active slot concept | 2-3 hrs |
| 4 | Near-cursor UI | Icons appear on Ctrl+C | 4-6 hrs |

**Success criteria:** 
- On Windows, Ctrl+C shows 9 slot icons near cursor
- Clicking a slot copies content there
- Clicking a slot during Ctrl+V pastes from it
- App runs in system tray

**Estimated effort:** 12-18 hours to functional Windows prototype

### Phase 1: Windows MVP

| Milestone | Features | Target |
|-----------|----------|--------|
| 0.1.0 | Clipboard capture, 9 slots, tray icon | Week 1 |
| 0.2.0 | Near-cursor UI, tap/hold gestures | Week 2 |
| 0.3.0 | SQLite persistence, crash recovery | Week 3 |
| 0.4.0 | Settings UI, autostart | Week 4 |

### Phase 2: Cross-Platform

| Milestone | Features | Target |
|-----------|----------|--------|
| 0.5.0 | macOS support | Week 5-6 |
| 0.6.0 | Linux X11 support | Week 7-8 |
| 0.7.0 | Linux Wayland (degraded) | Week 9 |

### Phase 3: Polish & Launch

| Milestone | Features | Target |
|-----------|----------|--------|
| 0.8.0 | Content preview, animations | Week 10 |
| 0.9.0 | Security (sensitive data detection) | Week 11 |
| 1.0.0 | Code signing, auto-update, website | Week 12 |

### Post-1.0 Backlog (Not Committed)

| Feature | Complexity | Value | Notes |
|---------|------------|-------|-------|
| Image/file support | Medium | High | v1.1 candidate |
| Search across history | Medium | Medium | Maybe v1.2 |
| Snippet library | High | Medium | Separate feature, post-1.0 |
| Encrypted storage | Medium | Low | Optional, for enterprise |

---

## 4) Non-Goals (What we are NOT doing)

| Feature | Reason |
|---------|--------|
| Cloud sync | Privacy principle; complexity; scope creep |
| Mobile app | Different paradigm; clipboard works differently |
| Browser extension | Separate product; security nightmare |
| Team features | Enterprise complexity; not our market |
| Plugin system | Premature abstraction; maintenance burden |
| Subscription | Competitive differentiation; user trust |
| Telemetry | Privacy principle; not needed |

---

## 5) Success Metrics

### Technical

| Metric | Target | Measurement |
|--------|--------|-------------|
| Cold start | <500ms | Time from launch to responsive |
| Tap latency | <10ms | Added latency to native Ctrl+C/V |
| Memory (idle) | <20MB | No content in slots |
| Memory (full) | <100MB | All slots occupied |
| Crash rate | <0.1% | Per session |

### Product

| Metric | Target | Measurement |
|--------|--------|-------------|
| Time to first slot use | <60s | From install to using slot 2+ |
| Feature discovery | >50% | Users who use hold-gesture in first week |
| Refund rate | <5% | Of purchases |
| Support tickets | <1% | Of purchases |

### Business

| Metric | Target | Measurement |
|--------|--------|-------------|
| Conversion rate | >3% | Trial to purchase |
| CAC | <$5 | Customer acquisition cost |
| LTV | $12 | One-time purchase, no churn math needed |

---

## 6) Instructions for Implementation

**To the next session (or future me):**

Your top priority is Phase 0-4 of "Immediate Next Action."

1. Read `docs/ARCHITECTURE.md` completely
2. Scaffold workspace with `cargo new --lib` for each crate
3. Get clipboard monitoring working on Windows first
4. Add system tray second
5. Add near-cursor UI third
6. Test the feel—timing is everything

**Critical files to create first:**
```
pockets/
├── Cargo.toml              # Workspace
├── crates/
│   ├── pockets-core/       # Slots, events, config
│   ├── pockets-clipboard/  # Platform abstraction
│   ├── pockets-hotkey/     # Gesture detection
│   ├── pockets-persist/    # SQLite
│   └── pockets-ui/         # Dioxus
└── src/
    └── main.rs             # Entry point
```

**First milestone:** `cargo run` opens a system tray icon that shows "Pockets" tooltip.

---

## Changelog

| Date | Change |
|------|--------|
| 2025-12-30 | Initial architecture complete. Ready for implementation. |
