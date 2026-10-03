# PyBlockWorld macOS Retina Rendering Fix

A reproducible fix for incomplete OpenGL rendering in PyBlockWorld on macOS Retina displays.

## Problem

When running PyBlockWorld on a Mac with a Retina display, the game window can render correctly on only part of the window. The remaining area appears as a uniform sky-blue background.

The issue was reproduced on:

* MacBook Air with Apple M1
* macOS
* Python 3.13.12
* PyBlockWorld 0.3.10
* pyglet 1.5.31

The same rendering test worked correctly in a standalone pyglet/OpenGL program, which isolated the problem to PyBlockWorld's viewport configuration.

## Symptoms

The PyBlockWorld window opens normally, but only part of the scene is rendered.

Typical appearance:

* left portion of the window renders normally
* right portion remains uniformly sky blue
* the application otherwise runs normally

## Root Cause

The problem is caused by the difference between the logical window size and the physical OpenGL framebuffer size on a Retina display.

For example, pyglet reported:

```text
window: 800 600
framebuffer: (1600, 1200)
```

The PyBlockWorld code used:

```python
width, height = self.get_size()
glViewport(0, 0, width, height)
```

On a Retina display, `get_size()` returns the logical window dimensions, while OpenGL's viewport needs to cover the physical framebuffer.

Therefore, PyBlockWorld configured the viewport as:

```text
800 × 600
```

while the actual framebuffer was:

```text
1600 × 1200
```

Only part of the framebuffer was consequently rendered.

## Fix

Use `get_framebuffer_size()` for the OpenGL viewport while continuing to use the logical window dimensions for the projection calculations.

The relevant change is:

```python
width, height = self.get_size()
fb_width, fb_height = self.get_framebuffer_size()
glViewport(0, 0, fb_width, fb_height)
```

This change needs to be made in both:

* `set_2d()`
* `set_3d()`

The projection calculations should continue to use `width` and `height`.

## Installation

Create a Python virtual environment and install the tested versions:

```bash
python3.13 -m venv pyblockworld-env
source pyblockworld-env/bin/activate
python -m pip install --upgrade pip
python -m pip install pyblockworld==0.3.10 pyglet==1.5.31
```

Then apply the patch in this repository to the installed PyBlockWorld package.

See `PATCH.md` for the detailed patch instructions.

## Minimal Test Program

The rendering problem can be reproduced with:

```python
from pyblockworld import World

world = World()
world.run()
```

This repository includes the example as `base.py`.

## Verification

A standalone pyglet/OpenGL test was also used to verify that Retina framebuffer handling itself was working correctly.

The test created an 800 × 600 logical window and reported:

```text
window: 800 600
framebuffer: (1600, 1200)
```

The entire window rendered correctly.

This established that the problem was not a general macOS, Apple Silicon, Retina, or pyglet OpenGL failure.

## Technical Explanation

There are two relevant coordinate spaces:

### Logical window size

Returned by:

```python
self.get_size()
```

Example:

```text
800 × 600
```

These dimensions are appropriate for the application's logical coordinate system and projection calculations.

### Physical framebuffer size

Returned by:

```python
self.get_framebuffer_size()
```

Example:

```text
1600 × 1200
```

These dimensions represent the actual pixel dimensions of the OpenGL framebuffer.

`glViewport()` operates on the framebuffer, so it needs the physical framebuffer dimensions.

The correct separation is therefore:

```python
width, height = self.get_size()
fb_width, fb_height = self.get_framebuffer_size()

glViewport(0, 0, fb_width, fb_height)
```

while retaining:

```python
glOrtho(0, width, 0, height, -1, 1)
```

and:

```python
gluPerspective(65.0, width / float(height), 0.1, 60.0)
```

## Why This Repository Exists

The purpose of this repository is to document the investigation and provide a small, reproducible fix rather than uploading a complete Python environment.

It intentionally does not contain:

* Python virtual environments
* installed packages
* `__pycache__` directories
* compiled Python files
* macOS metadata

## Files

```text
pyblockworld-macos-retina-fix/
├── README.md
├── PATCH.md
├── base.py
├── requirements.txt
├── retina-viewport.patch
└── .gitignore
```

## Upstream

This repository documents a specific rendering issue and its fix. It is intended to make the problem easier to reproduce and potentially useful for an upstream PyBlockWorld fix.

The fix is deliberately limited to the OpenGL viewport dimensions and does not change the application's logical window dimensions.
