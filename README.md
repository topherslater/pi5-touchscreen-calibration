# Touchscreen Calibration on Raspberry Pi (Wayland/Wayfire)

A guide for calibrating an ILI210x (or similar I2C/USB) touchscreen on a Raspberry Pi running Wayland with Wayfire — especially when the display is rotated for portrait/kiosk mode.

## The Problem

When running a Pi with a rotated display (e.g., portrait mode for a Home Assistant kiosk), the touchscreen coordinates often don't match the display. Symptoms include:

- Touch registers in the wrong location
- Touch only works at screen edges
- Gestures work (pinch-to-zoom) but taps don't click anything
- Axes are swapped or inverted

## Why It's Tricky

1. **Multiple rotation layers can fight each other:** `display_rotate` in config.txt, `wlr-randr` transforms, compositor `touchscreen_output` mapping, and `libinput` calibration matrices all affect touch mapping independently.

2. **Touch panels often use only a fraction of their reported coordinate range.** A panel might report a range of 0–16383 but only physically use 31–1919. Without correcting this, the entire screen maps to a tiny sliver of the coordinate space.

3. **`libinput debug-events` shows post-calibration values**, not raw hardware values. This can lead you down the wrong path if you use those numbers to calculate your calibration matrix.

## Prerequisites

- Raspberry Pi running Raspberry Pi OS Bookworm (or later) with Wayland/Wayfire
- A connected touchscreen (this guide uses ILI210x but applies to most evdev touch devices)
- Root/sudo access

## Step 1: Identify Your Touch Device

```bash
cat /proc/bus/input/devices | grep -B2 -A5 -i touch
```

Note the event device (e.g., `/dev/input/event5`) and the device name.

Verify with:
```bash
sudo libinput list-devices | grep -A5 "Touchscreen"
```

## Step 2: Capture TRUE Raw Coordinates

> **⚠️ Critical:** Do NOT use `libinput debug-events` for this step. It shows post-calibration values, not raw hardware values.

First, set an identity calibration matrix (no transformation):
```bash
sudo bash -c 'echo "ENV{ID_INPUT_TOUCHSCREEN}==\"1\", ENV{LIBINPUT_CALIBRATION_MATRIX}=\"1 0 0 0 1 0 0 0 1\"" > /etc/udev/rules.d/99-touchscreen.rules'
sudo udevadm control --reload-rules
sudo udevadm trigger
```

Remove any existing hwdb overrides:
```bash
sudo rm -f /etc/udev/hwdb.d/99-touchscreen.hwdb
sudo systemd-hwdb update
```

Reboot, then run this Python script to capture raw coordinates. Replace `event5` with your device:

```bash
sudo timeout 30 python3 -c "
import struct, os, time, select

DEVICE = '/dev/input/event5'
fd = os.open(DEVICE, os.O_RDONLY | os.O_NONBLOCK)
print('READY — tap 4 corners: TL, TR, BR, BL (3-second pause between each)', flush=True)

x, y = 0, 0
taps = []
last_tap = 0
start = time.time()

while time.time() - start < 28:
    r, _, _ = select.select([fd], [], [], 0.1)
    if r:
        try:
            data = os.read(fd, 24)
            while len(data) >= 24:
                chunk = data[:24]
                data = data[24:]
                tv_sec, tv_usec, typ, code, value = struct.unpack('llHHi', chunk)
                if typ == 3:  # EV_ABS
                    if code in (0, 53): x = value   # ABS_X or ABS_MT_POSITION_X
                    elif code in (1, 54): y = value  # ABS_Y or ABS_MT_POSITION_Y
                elif typ == 1 and code == 330 and value == 1:  # BTN_TOUCH down
                    now = time.time()
                    if now - last_tap > 1:  # Deduplicate rapid events
                        taps.append((x, y))
                        print(f'Tap {len(taps)}: X={x} Y={y}', flush=True)
                        last_tap = now
        except BlockingIOError:
            pass

os.close(fd)
print(f'\\nTotal taps: {len(taps)}')
print(f'X range: {min(t[0] for t in taps)} — {max(t[0] for t in taps)}')
print(f'Y range: {min(t[1] for t in taps)} — {max(t[1] for t in taps)}')
"
```

You should get output like:
```
Tap 1: X=1919 Y=27      (top-left)
Tap 2: X=33   Y=1062    (top-right — or whichever corner)
Tap 3: X=31   Y=13      (bottom-right)
Tap 4: X=1910 Y=1055    (bottom-left)

X range: 31 — 1919
Y range: 13 — 1062
```

## Step 3: Set the Correct ABS Range (hwdb)

The raw coordinates tell you the actual physical range of the touch panel. Create a hwdb file to override the kernel's default (often wildly wrong) range:

```bash
sudo mkdir -p /etc/udev/hwdb.d
sudo tee /etc/udev/hwdb.d/99-touchscreen.hwdb << EOF
evdev:input:b0018v0000p0000*
 EVDEV_ABS_00=<X_MIN>:<X_MAX>
 EVDEV_ABS_01=<Y_MIN>:<Y_MAX>
 EVDEV_ABS_35=<X_MIN>:<X_MAX>
 EVDEV_ABS_36=<Y_MIN>:<Y_MAX>
EOF
```

Replace `<X_MIN>`, `<X_MAX>`, `<Y_MIN>`, `<Y_MAX>` with your values from Step 2.

**Finding the match string for your device:**

The format is `evdev:input:b<BUS>v<VENDOR>p<PRODUCT>e<VERSION>*`. Get these from:
```bash
cat /proc/bus/input/devices | grep -B2 -A5 -i touch
# Look for: Bus=0018 Vendor=0000 Product=0000 Version=0000
```

The `EVDEV_ABS` codes are:
- `00` = ABS_X
- `01` = ABS_Y
- `35` = ABS_MT_POSITION_X (multitouch)
- `36` = ABS_MT_POSITION_Y (multitouch)

Apply:
```bash
sudo systemd-hwdb update
```

## Step 4: Determine the Calibration Matrix

The calibration matrix is a 3×3 affine transformation applied to normalized touch coordinates (0–1):

```
[a  b  c]   [input_x]   [output_x]
[d  e  f] × [input_y] = [output_y]
[0  0  1]   [   1   ]   [   1    ]
```

### Common Rotation Matrices

| Rotation | Matrix | Use When |
|----------|--------|----------|
| None (identity) | `1 0 0 0 1 0 0 0 1` | Touch matches display |
| 90° CW | `0 1 0 -1 0 1 0 0 1` | Display rotated 270° (portrait) |
| 90° CCW | `0 -1 1 1 0 0 0 0 1` | Display rotated 90° |
| 180° | `-1 0 1 0 -1 1 0 0 1` | Display rotated 180° |

### How to Determine Which Matrix You Need

After setting the hwdb range (Step 3), map your raw corner coordinates to normalized values (0–1). Then figure out which screen position each corner maps to. The matrix that transforms touch corners to screen corners is what you need.

**Quick method:** Try each matrix, reboot, and test. Start with the one that matches your display rotation.

Apply:
```bash
sudo bash -c 'echo "ENV{ID_INPUT_TOUCHSCREEN}==\"1\", ENV{LIBINPUT_CALIBRATION_MATRIX}=\"0 1 0 -1 0 1 0 0 1\"" > /etc/udev/rules.d/99-touchscreen.rules'
sudo udevadm control --reload-rules
```

## Step 5: Configure Wayfire

### Display Rotation

Use `wlr-randr` in your Wayfire autostart — **not** `display_rotate` in config.txt (which doesn't work properly with Wayfire):

```ini
[autostart]
rotate_display = WAYLAND_DISPLAY=wayland-1 wlr-randr --output HDMI-A-1 --transform 270
```

If you had `display_rotate=N` in `/boot/firmware/config.txt`, comment it out:
```bash
sudo sed -i 's/^display_rotate=/#display_rotate=/' /boot/firmware/config.txt
```

### Do NOT Use `touchscreen_output`

When using a calibration matrix to handle touch rotation, **do not** set `touchscreen_output` in wayfire.ini. The compositor's transform will fight with your matrix, doubling the rotation:

```ini
# DON'T do this when using a calibration matrix:
# [input]
# touchscreen_output = HDMI-A-1
```

## Step 6: Reboot and Test

```bash
sudo reboot
```

After reboot, tap different areas of the screen and verify:
- Taps register at the correct position
- Scrolling works in the correct direction
- Buttons/links are clickable

## Troubleshooting

### Touch coordinates are compressed into one edge
Your hwdb range is wrong. Re-run the raw coordinate capture (Step 2) and verify you're reading TRUE raw values (Python script, not `libinput debug-events`).

### Taps don't click but gestures (pinch/scroll) work
The coordinates are likely out of the 0–1 normalized range. This happens when:
- hwdb range doesn't match actual hardware values
- A large-value calibration matrix amplifies small movements (turning taps into scrolls)

### Everything is mirrored/flipped
Try a different calibration matrix. There are only 4 rotation options plus axis flips — brute force is faster than math.

### Cursor not visible
Normal for Wayland touch input. Touch operates like a phone — direct tap, no cursor. This is ideal for kiosk setups.

### `libinput debug-events` shows crazy percentages (900%+)
The hwdb range is much smaller than the hardware's default range, and calibration values are post-transform. Use the Python script for raw values.

## Example: Working Config

**Hardware:** Raspberry Pi 5 + ILI210x I2C touchscreen + 1080p HDMI display in portrait mode (270° rotation)

**`/etc/udev/hwdb.d/99-touchscreen.hwdb`:**
```
evdev:input:b0018v0000p0000*
 EVDEV_ABS_00=31:1919
 EVDEV_ABS_01=13:1062
 EVDEV_ABS_35=31:1919
 EVDEV_ABS_36=13:1062
```

**`/etc/udev/rules.d/99-touchscreen.rules`:**
```
ENV{ID_INPUT_TOUCHSCREEN}=="1", ENV{LIBINPUT_CALIBRATION_MATRIX}="0 1 0 -1 0 1 0 0 1"
```

**`~/.config/wayfire.ini`:**
```ini
[autostart]
chromium = chromium-browser http://192.168.0.200:8123/lovelace/kitchen --kiosk --noerrdialogs --disable-infobars --no-first-run --ozone-platform=wayland --enable-features=OverlayScrollbar --start-maximized
rotate_display = WAYLAND_DISPLAY=wayland-1 wlr-randr --output HDMI-A-1 --transform 270
screensaver = false
dpms = false
```

**`/boot/firmware/config.txt`:**
```
# display_rotate=3  ← commented out, wlr-randr handles this
```

## Key Lessons Learned

1. **`display_rotate` in config.txt does NOT work with Wayfire** for application content — only affects the framebuffer/console. Use `wlr-randr` instead.

2. **`libinput debug-events` shows post-calibration coordinates.** Always use raw evdev reads (Python/C) when determining the touch panel's actual coordinate range.

3. **Don't mix compositor touch transforms with calibration matrices.** Use one or the other. The matrix approach gives you full control.

4. **`ts_calibrate` (tslib) does NOT affect Wayland/libinput.** It only works for tslib-based applications.

5. **Touch on Wayland has no visible cursor.** This is by design — direct touch input works like a phone screen.

---

*Tested on Raspberry Pi 5, Raspberry Pi OS Bookworm, Wayfire compositor, ILI210x I2C touchscreen, March 2026.*
