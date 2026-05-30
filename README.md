# plasma-panel-touch-hider

Fixes KDE Plasma panels staying open after a touchscreen edge swipe gesture.

## Problem

In KDE Plasma 6 (Wayland), panels set to **Dodge Windows** or **Auto-hide** mode can be revealed by swiping from the screen edge. However, when triggered by touch (rather than mouse), the panel never hides again - it stays visible until you click something. This is an architectural gap in the `kde-plasma-shell` Wayland protocol, which only defines pointer-based hiding triggers, with no equivalent for touch input.

## Solution

A small Python daemon reads raw touch events directly from the touchscreen (`/dev/input/eventX`, via `python-evdev`) in parallel with KWin, without interfering with it. When it detects a swipe gesture that started from the top or bottom edge, it waits 5 seconds, then triggers KWin to re-evaluate panel visibility via DBus. Panels hide only if a window is actually overlapping them - on the empty desktop they stay visible.

Mouse hover behavior is completely unaffected.

## Requirements

- KDE Plasma 6 (Wayland)
- `python-evdev`

## Installation

```bash
# Install dependency
sudo pacman -S python-evdev   # Arch/CachyOS
# or: pip install evdev

# Add yourself to the input group
sudo usermod -aG input $USER

# Copy files
cp plasma-panel-touch-hider ~/.local/bin/plasma-panel-touch-hider
chmod +x ~/.local/bin/plasma-panel-touch-hider
cp plasma-panel-touch-hider.service ~/.config/systemd/user/

# Enable and start
systemctl --user daemon-reload
systemctl --user enable --now plasma-panel-touch-hider.service
```

> **Note:** If you don't want to re-login for the group change to take effect, the included service file uses `sg input -c '...'` as a workaround. After re-login you can simplify the `ExecStart` line.

## Configuration

Edit `plasma-panel-touch-hider` and adjust at the top:

```python
DEVICE_NAME   = "Wacom Pen and multitouch sensor Finger"  # your touch device name
EDGE_FRACTION = 0.06   # swipe must start within 6% of screen edge
HIDE_DELAY    = 5.0    # seconds before hiding
```

Find your device name with:
```bash
sudo libinput list-devices
# or
cat /proc/bus/input/devices
```

## How it works

1. Reads `ABS_MT_TRACKING_ID` and `ABS_MT_POSITION_Y` events from the touchscreen device
2. Records the Y coordinate of each new touch contact
3. When the finger lifts (`TRACKING_ID = -1`), checks if it started near the top or bottom edge
4. If yes, schedules a 5-second timer
5. On timer fire, calls `qdbus6 org.kde.plasmashell evaluateScript` which toggles each panel's hiding mode between `dodgewindows` and `dodgeactive` - this triggers KWin to re-evaluate window positions from the current visible state, hiding the panel only if a window actually overlaps it

## Tested on

- KDE Plasma 6.6.5, Wayland
- Wacom Pen and multitouch sensor (ThinkPad built-in digitizer)
