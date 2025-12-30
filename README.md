# Pockets

Extra pockets for your clipboard. Copy more, lose nothing.

Pockets is a cross-platform clipboard manager that adds 9 slots to your clipboard without changing how copy and paste work. It watches your clipboard, remembers what you copy, and lets you pick which slot to paste from—all without leaving your keyboard.

**$12. One time. Forever.**

[Download for Windows](https://pockets.app/download/windows) · [Download for macOS](https://pockets.app/download/macos) · [Download for Linux](https://pockets.app/download/linux)

---

## How It Works

Pockets doesn't replace your clipboard. It sits alongside it.

When you copy, Pockets catches a copy. When you want to paste from a different slot, Pockets loads it into your clipboard, then you paste normally. The system never knows Pockets exists.

```
        SYSTEM CLIPBOARD              POCKETS
        ┌─────────────┐         ┌───┬───┬───┬───┐
        │  (stage)    │◄───────►│ 1 │ 2 │ 3 │...│
        └─────────────┘         └───┴───┴───┴───┘
              ▲                       ▲
              │                       │
         [Ctrl+V]              [You select a slot]
```

## Usage

### Zero Learning Curve (Ignore Everything)

If you never learn anything about Pockets:
- **Ctrl+C** works exactly like before
- **Ctrl+V** works exactly like before
- Pockets just remembers your last 9 copies

You can stop reading here and you'll still benefit.

### Basic (Click to Select)

When you press Ctrl+C or Ctrl+V, small icons appear near your cursor:

```
  [1]  [2]  [3]  ...
   ▲
   │
   └── Green = active slot (what Ctrl+V will paste)
       White = has content  
       Gray  = empty
```

- **Click a slot after Ctrl+C** → Copy goes to that slot
- **Click a slot after Ctrl+V** → Paste from that slot

Icons fade after 2 seconds. Hover to keep them visible.

### Power User (Keyboard Only)

**Tap + Number:**
```
Ctrl+C, then 3       → Copy to slot 3
Ctrl+V, then 5       → Paste from slot 5
```

**Hold for Picker:**
```
Hold Ctrl+C (1.5s)   → Picker appears, press 1-9 to copy there
Hold Ctrl+V (1.5s)   → Picker appears, press 1-9 to paste from there
```

### Quick Reference

| Action | Gesture |
|--------|---------|
| Copy to active slot | Ctrl+C (tap) |
| Copy to slot N | Ctrl+C, then N |
| Copy to slot N (alternate) | Ctrl+C (hold), then N |
| Paste from active slot | Ctrl+V (tap) |
| Paste from slot N | Ctrl+V, then N |
| Paste from slot N (alternate) | Ctrl+V (hold), then N |
| Change active slot | Click any slot icon |
| Cancel | Escape |

---

## Principles

### Ctrl+C and Ctrl+V Are Sacred

Pockets will never:
- Add latency to normal copy/paste
- Show dialogs that interrupt your flow
- Change behavior you've relied on for decades

A tap is a tap. It works exactly like native. The slot icons are informational, not mandatory.

### Privacy by Default

Pockets will never:
- Send your clipboard anywhere
- Phone home with analytics
- Require an account or login

Everything stays on your machine. There's no cloud. There's no sync. There's no "sign in to continue."

### Handles What Your Clipboard Handles

Text. Images. Files. HTML. Whatever you copy, Pockets stores. We don't modify content. We don't transcode images. We don't strip formatting unless you ask.

### Data Survives

Crashes, reboots, updates—your slots persist. We use write-ahead logging. We test crash recovery. Your data is yours.

---

## Installation

### Windows

Download the installer from [pockets.app](https://pockets.app/download/windows) or:

```powershell
winget install Pockets
```

### macOS

Download the DMG from [pockets.app](https://pockets.app/download/macos) or:

```bash
brew install --cask pockets
```

**Note:** Pockets requires Accessibility permission to detect keyboard shortcuts. macOS will prompt you on first launch. This is standard for any app that uses global hotkeys.

**Why not the Mac App Store?** Apple's sandbox restrictions prevent apps from monitoring the clipboard or detecting global keyboard shortcuts. These are core features. Direct download is the only option.

### Linux

**Flatpak (Recommended):**
```bash
flatpak install flathub app.pockets.Pockets
```

**AppImage:**
Download from [pockets.app](https://pockets.app/download/linux).

**Arch (AUR):**
```bash
yay -S pockets
```

**Wayland Note:** Global hotkeys don't work on most Wayland compositors due to security restrictions. On Wayland, use the system tray to access slots. X11 works normally.

---

## Configuration

Pockets works out of the box. Configuration is optional.

Config location:
- Windows: `%APPDATA%\Pockets\config.toml`
- macOS: `~/Library/Application Support/Pockets/config.toml`
- Linux: `~/.config/pockets/config.toml`

```toml
[general]
slot_count = 9           # 1-9
autostart = true         # Start on login
show_tray = true         # System tray icon

[timing]
tap_threshold = 300      # ms - max duration for "tap"
hold_threshold = 1500    # ms - hold duration for picker
icon_display = 2000      # ms - how long icons stay visible

[storage]
max_size_mb = 100        # Max database size

[security]
# Apps whose clipboard content should be ignored
excluded_apps = [
    "1Password",
    "Bitwarden",
    "KeePassXC",
]
```

---

## Security

### What We Capture

By default, Pockets captures everything you copy. This includes passwords if you copy them manually.

### What We Don't Capture

- Content marked as "concealed" by password managers (we honor `org.nspasteboard.ConcealedType` on macOS and equivalent markers)
- Content from apps in your exclusion list
- Content marked as "transient"

### Recommendations

1. Use your password manager's auto-type feature instead of copy/paste for passwords
2. Add your password manager to `excluded_apps` in config
3. For highly sensitive work, consider clearing slots periodically (right-click tray → Clear All)

### Storage

Slot content is stored in a SQLite database on your local machine. It is not encrypted by default. If you need encryption at rest, enable FileVault (macOS), BitLocker (Windows), or LUKS (Linux) for your drive.

---

## FAQ

**Q: Does Pockets slow down copy/paste?**

No. Pockets adds <10ms to clipboard operations. This is imperceptible. We monitor the clipboard asynchronously after the system clipboard operation completes.

**Q: Does Pockets work with images?**

Yes. Text, images, files, HTML, rich text—whatever your clipboard handles, Pockets handles.

**Q: Can I sync across devices?**

No. This is a deliberate choice. Cloud sync adds complexity, requires accounts, raises privacy concerns, and creates failure modes. There are other tools if you need sync.

**Q: Why 9 slots? Why not unlimited history?**

Nine fits on one hand (keyboard row 1-9). It's enough to be useful, few enough to remember. Unlimited history is a different product—a clipboard *history* manager. Pockets is a clipboard *augmenter*. Different mental model.

**Q: Why isn't this free?**

Because I want to maintain it for years. Free tools get abandoned. $12 funds ongoing development, code signing certificates, and my coffee habit.

**Q: Can I get a refund?**

Yes, within 30 days, no questions asked. Email refund@pockets.app.

**Q: Why isn't this on the Mac App Store?**

Apple's sandbox prevents clipboard monitoring and global hotkey detection. These are core features. Direct distribution is the only option for clipboard managers.

**Q: Is there a trial?**

Yes. Download and use Pockets free for 14 days. Full features, no restrictions.

**Q: Is this open source?**

Yes. MIT license. [github.com/pockets/pockets](https://github.com/pockets/pockets)

You can build from source for free. The $12 gets you:
- Signed binaries (no security warnings)
- Automatic updates
- Supporting continued development

---

## Building from Source

```bash
git clone https://github.com/pockets/pockets
cd pockets
cargo build --release
```

The binary will be at `target/release/pockets` (or `pockets.exe` on Windows).

**Dependencies:**
- Rust 1.75+
- Platform-specific dependencies (see `docs/BUILDING.md`)

---

## Contributing

Contributions welcome. Please read `CONTRIBUTING.md` first.

**Good first issues:**
- UI polish and animations
- Accessibility improvements
- Documentation
- Translations

**Before starting major work:**
Open an issue to discuss. This avoids wasted effort if the feature doesn't fit the project's scope.

---

## License

MIT. See `LICENSE`.

---

## Acknowledgments

- Larry Tesler (1945-2020), who invented cut, copy, and paste
- The Rust community
- Everyone who tested early builds

---

## Support

- **Bugs:** [GitHub Issues](https://github.com/pockets/pockets/issues)
- **Questions:** [GitHub Discussions](https://github.com/pockets/pockets/discussions)
- **Email:** hello@pockets.app

---

*Pockets: Because your clipboard should remember more than one thing.*
