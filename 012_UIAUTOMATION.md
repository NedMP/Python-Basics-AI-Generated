

# Python UI Automation (Keyboard & Mouse) — updated Oct 5, 2025

Beginner‑friendly guide to desktop UI automation on macOS using **PyAutoGUI** with supporting tools. Covers install, Accessibility permissions, coordinates and screen capture, mouse and keyboard control, hotkeys, clipboard, image matching, window management, waits, safety, and troubleshooting. Use with `002_SETUP.md` for environment setup.

---

**Assumptions and Conventions**
- macOS + zsh, Python 3.12+ or 3.13 in a venv.  
- Automation needs **System Settings → Privacy & Security → Accessibility** permission for your terminal or IDE.

---

## Table of Contents
- [0) Install and grant permissions](#0-install-and-grant-permissions)
- [1) Safety first: failsafe, pauses, dry runs](#1-safety-first-failsafe-pauses-dry-runs)
- [2) Screen geometry and coordinates](#2-screen-geometry-and-coordinates)
- [3) Mouse control](#3-mouse-control)
- [4) Keyboard input and hotkeys](#4-keyboard-input-and-hotkeys)
- [5) Clipboard & typing helpers](#5-clipboard--typing-helpers)
- [6) Image recognition: locate on screen](#6-image-recognition-locate-on-screen)
- [7) Window management](#7-window-management)
- [8) Waiting for UI states](#8-waiting-for-ui-states)
- [9) Multi‑display, Retina scaling, and coordinates](#9-multi-display-retina-scaling-and-coordinates)
- [10) Project structure and a tiny macro](#10-project-structure-and-a-tiny-macro)
- [11) Alternatives and when to use them](#11-alternatives-and-when-to-use-them)
- [12) Troubleshooting](#12-troubleshooting)
- [13) Recap](#13-recap)

---

## 0) Install and grant permissions
**What**: Install packages and enable Accessibility control so Python can move the mouse and send keys.
```sh
pip install pyautogui pyobjc pillow pyperclip
```
Grant control:
1. Open **System Settings → Privacy & Security → Accessibility**.  
2. Add your terminal (iTerm/Terminal.app) and editor (VS Code) and **enable** them.  
3. Restart those apps after toggling permissions.

Quick verify:
```python
import pyautogui as pag
print(pag.size())     # screen width, height
print(pag.position()) # current mouse position
```

---

## 1) Safety first: failsafe, pauses, dry runs
**Why**: Automation can misfire. Keep an escape hatch.
```python
import pyautogui as pag
pag.FAILSAFE = True            # moving mouse to top‑left aborts
pag.PAUSE = 0.2                # add delay after each call
```
Add your own guard:
```python
try:
    # actions
    pass
except pag.FailSafeException:
    print("Aborted by moving mouse to top‑left corner")
```
For dry runs, print target coordinates before acting.

---

## 2) Screen geometry and coordinates
**What**: Understand pixel coordinates and regions.
```python
import pyautogui as pag
w, h = pag.size()
print(w, h)

x, y = pag.position()
print(x, y)

# Screenshot to inspect pixels
shot = pag.screenshot("desktop.png")
```
Tip: Use `pag.displayMousePosition()` helper (run from shell) to live‑print mouse coords and pixel color.

---

## 3) Mouse control
```python
import pyautogui as pag

pag.moveTo(100, 200, duration=0.3)
pag.dragTo(400, 200, duration=0.5)
pag.click(400, 200, button="left")
pag.doubleClick()
pag.rightClick()

# Scroll lines (positive up)
pag.scroll(600)

# Click relative to current position
pag.move(50, 0); pag.click()
```
Use `duration` for human‑like motion and to observe actions.

---

## 4) Keyboard input and hotkeys
```python
import pyautogui as pag

pag.typewrite("Hello, world!", interval=0.02)
pag.press("enter")
pag.hotkey("command", "space")   # Spotlight

# special keys: 'enter','esc','tab','delete','backspace','up','down','left','right'
# modifiers: 'command','ctrl','option','shift'
```
On Windows/Linux use `"ctrl"` instead of macOS `"command"`.

---

## 5) Clipboard & typing helpers
```python
import pyautogui as pag, pyperclip
text = "Nín hǎo, 世界"  # unicode safe via clipboard
pyperclip.copy(text)
pag.hotkey("command", "v")
```
Clipboard avoids layout issues and IME quirks.

---

## 6) Image recognition: locate on screen
**What**: Find buttons/icons by template matching.  
**Why**: Works when there’s no reliable keyboard navigation.
```python
import pyautogui as pag

box = pag.locateOnScreen("img/submit.png", confidence=0.9)  # requires OpenCV if using confidence
if box:
    center = pag.center(box)
    pag.moveTo(center)
    pag.click()
else:
    print("submit button not found")
```
Tips:
- Take crisp screenshots at native scale.  
- Set `confidence=0.8..0.95` (install `opencv-python`).  
- Restrict search to a **region** for speed: `region=(x, y, w, h)`.

---

## 7) Window management
Basic window queries via PyAutoGUI are limited. Use **pygetwindow** for titles and moves.
```sh
pip install pygetwindow
```
```python
import pygetwindow as gw
wins = gw.getWindowsWithTitle("Notes")
if wins:
    w = wins[0]
    w.activate(); w.moveTo(50, 50); w.resizeTo(1200, 800)
```

---

## 8) Waiting for UI states
Avoid `sleep()` only. Wait for conditions.
```python
import time, pyautogui as pag

# wait until an image appears or timeout
start = time.time()
while time.time() - start < 10:
    box = pag.locateOnScreen("img/done.png", confidence=0.9)
    if box:
        pag.click(pag.center(box)); break
    time.sleep(0.25)
else:
    raise TimeoutError("Timed out waiting for completion")
```

---

## 9) Multi‑display, Retina scaling, and coordinates
- Coordinate origin is top‑left of the **main** display.  
- On Retina, PyAutoGUI handles scaling, but mismatched template DPI breaks image matching. Capture templates on the same display.  
- For external monitors, confirm positions with `displayMousePosition()`.

---

## 10) Project structure and a tiny macro
```
uia-demo/
├── .venv/
├── img/
│   └── submit.png
├── macro.py
└── requirements.txt
```
`macro.py` example: open Spotlight, type an app name, and click a button once the app loads.
```python
import time
import pyautogui as pag

pag.FAILSAFE = True
pag.PAUSE = 0.2

# 1) Launch app via Spotlight
pag.hotkey("command", "space")
pag.typewrite("Notes\n", interval=0.02)

# 2) Wait for app to be ready
start = time.time()
while time.time() - start < 15:
    box = pag.locateOnScreen("img/submit.png", confidence=0.9)
    if box:
        pag.click(pag.center(box)); break
    time.sleep(0.3)
```

---

## 11) Alternatives and when to use them
- **pynput**: low‑level keyboard/mouse hooks and listeners. Good for hotkey daemons.  
- **keyboard / mouse**: Windows‑centric; limited on macOS without extra perms.  
- **App‑specific automation**: Prefer real APIs when available: AppleScript/JXA, Accessibility APIs, or app CLIs.  
- **Browser automation**: Use Playwright/Selenium instead of UI automation for web tasks.

Decision rule: choose **semantic** automation (APIs, CLIs) before pixel automation. Use PyAutoGUI only when no better hook exists.

---

## 12) Troubleshooting
- **Nothing happens**: missing Accessibility permission. Re‑add Terminal/VS Code in System Settings and restart them.  
- **Wrong clicks on Retina**: capture images on the same display and ensure scaling 100%.  
- **`ImageNotFoundException`**: check file paths; use absolute paths from `__file__`.  
- **Lag or stutter**: remove excessive screenshots; add `region=` and lower confidence.  
- **Stuck automation**: move mouse to top‑left to trigger FAILSAFE or `Ctrl+C` in terminal.  
- **Different layouts**: prefer clipboard paste for non‑ASCII or IME input.

---

## 13) Recap
```plaintext
Install libs → grant Accessibility → set failsafe/pause → read screen size → move/click/scroll → type/hotkeys → locate by image → wait for states → handle multi‑display → prefer APIs over pixels
```

**Next**: Add structured logging, screenshots on failure, and parameterize coordinates via a config file. Consider writing small, idempotent actions and composing them into reliable workflows.