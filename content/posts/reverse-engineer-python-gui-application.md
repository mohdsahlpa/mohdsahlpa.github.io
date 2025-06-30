---
title: "Reverse Engineering a Python GUI Application Built with PyInstaller"
date: 2025-06-30
author: "mohdsahlpa"
description: "A step-by-step deep dive into reverse engineering a PyInstaller-packed Python GUI executable to recover its source code."
tags: ["reverse engineering", "pyinstaller", "python", "security"]
---

> _Disclaimer: This blog post is meant strictly for educational and ethical security research. Do not reverse engineer software without permission._

## 🧠 Introduction

Reverse engineering Python applications can be fascinating, especially when they’re bundled into `.exe` files using tools like PyInstaller. In this post, I'll walk you through how I deconstructed a Windows executable (`app.exe`), identified it as a PyInstaller package, and recovered the original `app.py` source code.

---

## 🔍 Step 1: Passive Reconnaissance

I started with no source just a single `app.exe` binary.

### checking file properties

first, I ran the classic:

```bash
$ file app.exe # returns the type of file
app.exe: PE32+ executable (GUI) x86-64, for MS Windows, 7 sections
```
now we know that this is a windows executable file.

we could just use strings to confirm weather it is a pyinstaller package or not:

```bash
$ strings app.exe | grep pyinstaller # should return strings matching pyinstaller
_pyinstaller_pyz
```
this confirms that it is a pyinstaller bundled python application.

## 🔧 Step 2: Extraction

### using an extractor

these pyinstaller bundled `.exe` files are like a zip file which we could just extract using tools like [pyinstxtractor](https://github.com/extremecoders-re/pyinstxtractor/).

```bash
$ python3 pyinstxtractor.py app.exe
[+] Processing app.exe
[+] Pyinstaller version: 2.1+
[+] Python version: 3.10
[+] Length of package: 59743120 bytes
[+] Found 1008 files in CArchive
[+] Beginning extraction...please standby
[+] Possible entry point: pyiboot01_bootstrap.pyc
[+] Possible entry point: pyi_rth_inspect.pyc
[+] Possible entry point: pyi_rth_pkgutil.pyc
[+] Possible entry point: pyi_rth_multiprocessing.pyc
[+] Possible entry point: pyi_rth__tkinter.pyc
[+] Possible entry point: exe.pyc
[!] Warning: This script is running in a different Python version than the one used to build the executable.
[!] Please run this script in Python 3.10 to prevent extraction errors during unmarshalling
[!] Skipping pyz extraction
[+] Successfully extracted pyinstaller archive: app.exe
You can now use a python decompiler on the pyc files within the extracted directory
```
now we have a folder named `app.exe_extracted` in our working directory and will contain the entire python source bytecode which we could process further. Also this output contains metadata about python and pyinstaller versions which will come handy during decompilation.

in the `app.exe_extracted` folder we have the following files(entire python bytecode):
```bash
PIL                 _lzma.pyd             cv2                    pyi_rth_inspect.pyc          select.pyd
PYZ.pyz             _multiprocessing.pyd  app.pyc                pyi_rth_multiprocessing.pyc  struct.pyc
PYZ.pyz_extracted   _overlapped.pyd       libcrypto-1_1.dll      pyi_rth_pkgutil.pyc          tcl8
VCRUNTIME140.dll    _queue.pyd            libffi-7.dll           pyiboot01_bootstrap.pyc      tcl86t.dll
VCRUNTIME140_1.dll  _socket.pyd           libssl-1_1.dll         pyimod01_archive.pyc         tk86t.dll
_asyncio.pyd        _ssl.pyd              numpy                  pyimod02_importers.pyc       unicodedata.pyd
_bz2.pyd            _tcl_data             numpy-2.2.6.dist-info  pyimod03_ctypes.pyc          win32
_ctypes.pyd         _tk_data              numpy.libs             pyimod04_pywin32.pyc         yaml
_decimal.pyd        _tkinter.pyd          psutil                 python3.dll
_elementtree.pyd    base_library.zip      pyexpat.pyd            python310.dll
_hashlib.pyd        charset_normalizer    pyi_rth__tkinter.pyc   pywin32_system32
```
here we can see the entry/main python file named `app.pyc` which is the file we have to decompile.

### decompilation

we can use a tool like [pycdc](https://github.com/zrax/pycd/) to decompile the python bytecode as it supports python version `3.10`.

```bash
$ pycdc ./app.exe_extracted/exe.pyc > app.py
```

voila!, now we have the source code in the app.py file which is:
```python
# Source Generated with Decompyle++
# File: app.pyc (Python 3.10)

import cv2
import threading
import tkinter as tk
from tkinter import messagebox
from PIL import Image, ImageTk

class App:
    
    def __init__(self, window):
        self.window = window
        self.window.title('YOLOv8n Person Detection')
        self.window.geometry('800x640')
        self.video_label = tk.Label(window)
        self.video_label.pack()
        button_frame = tk.Frame(window)
        button_frame.pack(10, **('pady',))
        self.start_btn = tk.Button(button_frame, 'Start Streaming', ('Arial', 12), self.start_thread, **('text', 'font', 'command'))
        self.start_btn.pack(tk.LEFT, 10, **('side', 'padx'))
        self.stop_btn = tk.Button(button_frame, 'Stop Streaming', ('Arial', 12), self.stop_streaming, tk.DISABLED, **('text', 'font', 'command', 'state'))
        self.stop_btn.pack(tk.LEFT, 10, **('side', 'padx'))
        self.cap = None
        self.streaming = False

    
    def start_thread(self):
        if not self.streaming:
            self.streaming = True
            self.stop_btn.config(tk.NORMAL, **('state',))
            self.start_btn.config(tk.DISABLED, **('state',))
            threading.Thread(self.stream, **('target',)).start()
            return None

    
    def stream(self):
        self.cap = cv2.VideoCapture(0)
        if not self.cap.isOpened():
            messagebox.showerror('Error', '❌ Cannot open video stream.')
            self.reset_buttons()
            return None
        if None.streaming:
            (ret, frame) = self.cap.read()
            if not ret:
                pass
            else:
                frame_rgb = cv2.cvtColor(frame, cv2.COLOR_BGR2RGB)
                img = Image.fromarray(frame_rgb)
                imgtk = ImageTk.PhotoImage(img, **('image',))
                self.video_label.imgtk = imgtk
                self.video_label.config(imgtk, **('image',))
                if not self.streaming:
                    self.cap.release()
                    self.reset_buttons()
                    return None

    
    def stop_streaming(self):
        self.streaming = False
        if self.cap:
            self.cap.release()
        self.window.after(100, self.window.destroy)

    
    def reset_buttons(self):
        self.start_btn.config(tk.NORMAL, **('state',))
        self.stop_btn.config(tk.DISABLED, **('state',))

    
    def on_close(self):
        self.streaming = False
        if self.cap:
            self.cap.release()
        self.window.destroy()


root = tk.Tk()
app = App(root)
root.protocol('WM_DELETE_WINDOW', app.on_close)
root.mainloop()
``` 

this looks like an application that uses your webcam to display a real-time video feed with OpenCV.

## 📦 Lessons To Note
- pyInstaller doesn't truly compile it's packages.
- security through obscurity isn’t real security.
- tools like `pyinstxtractor`, `pycdc` are powerful allies for binary introspection.
- always check the Python version before decompiling.

## 📌 Final Thoughts

reverse engineering can be easy with the help of AI tools like chatgpt and likely tooling but still requires a methodical approach.

always make sure you’re authorized to reverse engineer the application you’re looking into. This stuff gets legally gray real fast.