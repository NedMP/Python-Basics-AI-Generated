

# Python Desktop UI with Tkinter — updated Oct 5, 2025

Beginner‑friendly guide to building simple desktop GUIs with the Python standard library **tkinter** and themed widgets **ttk**. Covers window setup, layout managers, core widgets, variables and events, dialogs, menus, canvas, images, threading, and packaging. Use with `002_SETUP.md` for environment setup.

---

**Assumptions and Conventions**
- macOS + zsh, Python 3.12+ or 3.13 from Homebrew or python.org.  
- `tkinter` ships with Python. If import fails, reinstall Python (Homebrew `brew install python`) to get a recent Tk.

---

## Table of Contents
- [0) What is Tkinter](#0-what-is-tkinter)
- [1) Quick start: a window and a button](#1-quick-start-a-window-and-a-button)
- [2) Tk vs Ttk: themed widgets](#2-tk-vs-ttk-themed-widgets)
- [3) Layout managers: pack, grid, place](#3-layout-managers-pack-grid-place)
- [4) Variables and events](#4-variables-and-events)
- [5) Common widgets](#5-common-widgets)
- [6) Menus and dialogs](#6-menus-and-dialogs)
- [7) Canvas basics (drawing)](#7-canvas-basics-drawing)
- [8) Images and icons](#8-images-and-icons)
- [9) Structure a small app](#9-structure-a-small-app)
- [10) Long‑running work and the event loop](#10-long-running-work-and-the-event-loop)
- [11) Packaging options](#11-packaging-options)
- [12) Troubleshooting](#12-troubleshooting)
- [13) Recap](#13-recap)

---

## 0) What is Tkinter
**What**: Tkinter is Python’s built‑in binding to the Tk GUI toolkit.  
**Why**: Zero external deps, good for simple tools, forms, and small utilities. Cross‑platform (macOS/Windows/Linux).

Notes for macOS:
- Homebrew Python links against recent Tk. If fonts look odd or widgets are missing, update Python and try again.
- On Retina displays, Tk scales automatically; tweak via `tk.call('tk', 'scaling', factor)` if needed (e.g., `1.0`, `2.0`).

---

## 1) Quick start: a window and a button
**Goal**: Show a window with a label and a button that changes the label.

`app_basic.py`:
```python
import tkinter as tk
from tkinter import ttk

def on_click():
    label_var.set("Button clicked")

root = tk.Tk()
root.title("Tkinter Demo")
root.geometry("360x200")  # WxH in pixels

# Variables are observable values used by widgets
label_var = tk.StringVar(value="Hello, Tkinter!")

# Widgets
label = ttk.Label(root, textvariable=label_var)
btn = ttk.Button(root, text="Click me", command=on_click)

# Layout
label.pack(pady=16)
btn.pack()

root.mainloop()  # start event loop
```
Run:
```sh
python app_basic.py
```

---

## 2) Tk vs Ttk: themed widgets
Use **ttk** for modern‑looking widgets (Label, Button, Entry, Frame, Notebook, Treeview). They respect system themes.
```python
style = ttk.Style()
print(style.theme_names())     # list themes
style.theme_use("clam")        # or "aqua" on macOS, "alt", etc.
```
You can still use classic `tk.*` widgets when ttk lacks an option (e.g., `Text`, `Canvas`).

---

## 3) Layout managers: pack, grid, place
- **pack**: stack widgets vertically/horizontally. Simple UIs.  
- **grid**: table layout with rows/columns. Most flexible.  
- **place**: absolute positioning. Avoid for resizable apps.

`grid` example:
```python
root = tk.Tk()
root.title("Login")

frm = ttk.Frame(root, padding=16)
frm.grid(sticky="nsew")
root.columnconfigure(0, weight=1)
root.rowconfigure(0, weight=1)

# Form fields
user_var = tk.StringVar()
pass_var = tk.StringVar()

ttk.Label(frm, text="Username").grid(row=0, column=0, sticky="w", pady=4)
ttk.Entry(frm, textvariable=user_var).grid(row=0, column=1, sticky="ew", pady=4)

ttk.Label(frm, text="Password").grid(row=1, column=0, sticky="w", pady=4)
ttk.Entry(frm, textvariable=pass_var, show="•").grid(row=1, column=1, sticky="ew", pady=4)

frm.columnconfigure(1, weight=1)

btn = ttk.Button(frm, text="Sign in")
btn.grid(row=2, column=0, columnspan=2, pady=8)

root.mainloop()
```

---

## 4) Variables and events
Tkinter variables keep UI and code in sync.
```python
name = tk.StringVar(value="Ned")
age  = tk.IntVar(value=30)
check = tk.BooleanVar(value=False)
```
Trace changes:
```python
def on_name_change(*_):
    print("Name is now", name.get())
name.trace_add("write", on_name_change)
```
Bind events:
```python
entry = ttk.Entry(root, textvariable=name)
entry.bind("<Return>", lambda e: print("Enter pressed"))
root.bind("<Command-q>", lambda e: root.quit())   # macOS Cmd+Q
```

---

## 5) Common widgets
```python
# Labels and buttons
lbl = ttk.Label(root, text="Status: ready")
btn = ttk.Button(root, text="Run", command=lambda: print("go"))

# Text input
entry = ttk.Entry(root, textvariable=tk.StringVar())
text  = tk.Text(root, height=8, width=40)  # multiline

# Checkboxes, radios, combos
flag = tk.BooleanVar()
chk = ttk.Checkbutton(root, text="Enable", variable=flag)
opt = tk.StringVar(value="red")
r1  = ttk.Radiobutton(root, text="Red", value="red", variable=opt)
r2  = ttk.Radiobutton(root, text="Blue", value="blue", variable=opt)
combo = ttk.Combobox(root, values=["One","Two","Three"], state="readonly")

# Lists and trees
lst = tk.Listbox(root, height=5)
for i in range(5): lst.insert("end", f"Item {i}")

columns = ("name","size")
tree = ttk.Treeview(root, columns=columns, show="headings")
for c in columns:
    tree.heading(c, text=c.title())
tree.insert("", "end", values=("file.txt", "12 KB"))

# Notebook (tabs)
nb = ttk.Notebook(root)
nb.add(ttk.Frame(nb), text="Tab 1")
nb.add(ttk.Frame(nb), text="Tab 2")
```

---

## 6) Menus and dialogs
**Menus**
```python
menubar = tk.Menu(root)
filemenu = tk.Menu(menubar, tearoff=False)
filemenu.add_command(label="Open", command=lambda: print("open"))
filemenu.add_separator()
filemenu.add_command(label="Quit", command=root.quit)
menubar.add_cascade(label="File", menu=filemenu)
root.config(menu=menubar)
```
**Dialogs**
```python
from tkinter import messagebox, filedialog, colorchooser

messagebox.showinfo("Hello", "Operation complete")
path = filedialog.askopenfilename(title="Select file")
color = colorchooser.askcolor()[1]
```

---

## 7) Canvas basics (drawing)
Canvas lets you draw shapes and handle drag/zoom.
```python
cv = tk.Canvas(root, width=320, height=200, background="white")
cv.pack(fill="both", expand=True)
cv.create_rectangle(10, 10, 100, 80, outline="black", fill="#ddf")
cv.create_line(0, 0, 320, 200, width=2)
cv.create_text(160, 100, text="Canvas")
```
Bind interactions:
```python
def on_click(evt):
    cv.create_oval(evt.x-3, evt.y-3, evt.x+3, evt.y+3, fill="red", outline="")
cv.bind("<Button-1>", on_click)
```

---

## 8) Images and icons
Display images with `PhotoImage` (PNG/GIF) or Pillow for JPEG/modern formats.
```sh
pip install pillow
```
```python
from PIL import Image, ImageTk
img = Image.open("logo.png").resize((64,64))
photo = ImageTk.PhotoImage(img)
icon = ttk.Label(root, image=photo)
icon.image = photo  # prevent GC
icon.pack()

# Window icon (might not change on macOS dock)
root.iconphoto(True, photo)
```

---

## 9) Structure a small app
Use frames for screens and keep logic in methods.
```python
import tkinter as tk
from tkinter import ttk

class App(ttk.Frame):
    def __init__(self, master: tk.Misc):
        super().__init__(master, padding=16)
        master.title("Converter")
        master.minsize(360, 200)
        self.grid(sticky="nsew")
        master.columnconfigure(0, weight=1)
        master.rowconfigure(0, weight=1)

        self.kg = tk.DoubleVar(value=0.0)
        ttk.Label(self, text="kg").grid(row=0, column=0, sticky="w")
        ttk.Entry(self, textvariable=self.kg, width=10).grid(row=0, column=1)
        ttk.Button(self, text="Convert", command=self.convert).grid(row=0, column=2, padx=8)
        self.out = ttk.Label(self, text="lb: 0.00")
        self.out.grid(row=1, column=0, columnspan=3, sticky="w", pady=8)
        self.columnconfigure(1, weight=1)

    def convert(self):
        try:
            lbs = float(self.kg.get()) * 2.2046226218
            self.out.configure(text=f"lb: {lbs:.2f}")
        except ValueError:
            self.out.configure(text="Invalid number")

if __name__ == "__main__":
    root = tk.Tk()
    App(root)
    root.mainloop()
```

---

## 10) Long‑running work and the event loop
Tkinter is single‑threaded. Never block `mainloop()` with slow code. Use threads plus `after()` to update UI safely.
```python
import threading, time

def do_work():
    time.sleep(2)                 # simulate I/O or CPU work
    root.after(0, lambda: status.set("Done"))

status = tk.StringVar(value="Working…")
ttk.Label(root, textvariable=status).pack()
ttk.Button(root, text="Start", command=lambda: threading.Thread(target=do_work, daemon=True).start()).pack()
```
For periodic updates use `root.after(ms, callback)`. For asyncio, use `asyncio.get_event_loop().run_in_executor` or a bridge library; native Tkinter + asyncio integration is limited.

---

## 11) Packaging options
Make a single‑file app for testers.
- **pyinstaller**: quick bundling.
- **briefcase** or **py2app**: platform‑specific apps.
```sh
pip install pyinstaller
pyinstaller --windowed --onefile app_basic.py
```
Note: bundling Tcl/Tk inflates size. Test on a clean Mac before distributing.

---

## 12) Troubleshooting
- **`ImportError: No module named tkinter`**: reinstall Python from Homebrew or python.org; ensure Tk is bundled.
- **Blank window or UI freeze**: your handler blocks the loop. Move work to a thread and post results with `after()`.
- **Fonts too small/large on Retina**: adjust scaling `root.tk.call('tk','scaling', 2.0)`.
- **Images not showing**: keep a reference to `PhotoImage` (`widget.image = photo`).
- **Widget doesn’t resize**: configure row/column weights on container frames.

---

## 13) Recap
```plaintext
Create root → choose ttk theme → lay out with grid/pack → bind variables/events → add menus/dialogs → draw on canvas → load images → keep the loop responsive → package when ready
```

**Next**: Explore `ttk.Treeview` for tables, `ttk.Notebook` for tabs, and custom styles via `ttk.Style().configure()` to polish your app.