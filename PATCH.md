# PyBlockWorld Retina Viewport Patch

## Tested Environment

The fix was tested with:

* macOS on Apple Silicon
* Python 3.13.12
* PyBlockWorld 0.3.10
* pyglet 1.5.31

## Investigation

The original PyBlockWorld source configured the OpenGL viewport using the logical window dimensions:

```python
width, height = self.get_size()
glViewport(0, 0, width, height)
```

On a Retina display, pyglet can report different logical and physical dimensions.

The test environment reported:

```text
logical window: 800 × 600
physical framebuffer: 1600 × 1200
```

A standalone pyglet/OpenGL program using the same display rendered correctly across the entire window. This ruled out a general OpenGL, pyglet, or Retina rendering problem.

The difference was therefore traced to PyBlockWorld's viewport configuration.

## Original Code

The relevant parts of `pyblockworld/__init__.py` were:

```python
def set_2d(self):
    """ Configure OpenGL to draw in 2d.

    """
    width, height = self.get_size()
    glDisable(GL_DEPTH_TEST)
    glViewport(0, 0, width, height)
    glMatrixMode(GL_PROJECTION)
    glLoadIdentity()
    glOrtho(0, width, 0, height, -1, 1)
    glMatrixMode(GL_MODELVIEW)
    glLoadIdentity()
```

and:

```python
def set_3d(self):
    """ Configure OpenGL to draw in 3d.

    """
    width, height = self.get_size()
    glEnable(GL_DEPTH_TEST)
    glViewport(0, 0, width, height)
    glMatrixMode(GL_PROJECTION)
    glLoadIdentity()
    gluPerspective(65.0, width / float(height), 0.1, 60.0)
    glMatrixMode(GL_MODELVIEW)
    glLoadIdentity()
```

## Modified Code

The fix obtains the physical framebuffer dimensions separately:

```python
def set_2d(self):
    """ Configure OpenGL to draw in 2d.

    """
    width, height = self.get_size()
    fb_width, fb_height = self.get_framebuffer_size()
    glDisable(GL_DEPTH_TEST)
    glViewport(0, 0, fb_width, fb_height)
    glMatrixMode(GL_PROJECTION)
    glLoadIdentity()
    glOrtho(0, width, 0, height, -1, 1)
    glMatrixMode(GL_MODELVIEW)
    glLoadIdentity()
```

and:

```python
def set_3d(self):
    """ Configure OpenGL to draw in 3d.

    """
    width, height = self.get_size()
    fb_width, fb_height = self.get_framebuffer_size()
    glEnable(GL_DEPTH_TEST)
    glViewport(0, 0, fb_width, fb_height)
    glMatrixMode(GL_PROJECTION)
    glLoadIdentity()
    gluPerspective(65.0, width / float(height), 0.1, 60.0)
    glMatrixMode(GL_MODELVIEW)
    glLoadIdentity()
    x, y = self.rotation
    glRotatef(x, 0, 1, 0)
    glRotatef(-y, math.cos(math.radians(x)), 0, math.sin(math.radians(x)))
    x, y, z = self.position
    glTranslatef(-x, -y, -z)
```

## Why This Works

`get_size()` describes the logical window coordinate system.

`get_framebuffer_size()` describes the physical pixel dimensions of the framebuffer used by OpenGL.

`glViewport()` needs to cover the physical framebuffer, so it should use:

```python
fb_width, fb_height = self.get_framebuffer_size()
glViewport(0, 0, fb_width, fb_height)
```

The logical dimensions should still be used for the application's projection calculations.

This preserves the existing coordinate system while ensuring that OpenGL renders across the complete framebuffer.

## Result

After applying the change, the original PyBlockWorld test program rendered correctly across the complete window on the Retina display.

No changes to macOS, pyglet, OpenGL, or other applications were required.

## Scope

The change is limited to the PyBlockWorld installation being modified.

It does not modify:

* macOS
* the display configuration
* other Python installations
* other virtual environments
* other Python packages

A package reinstall or upgrade can overwrite the modified `__init__.py`, so the patch may need to be reapplied.

## Patch Summary

Conceptually, the fix changes these two lines:

```diff
- glViewport(0, 0, width, height)
+ fb_width, fb_height = self.get_framebuffer_size()
+ glViewport(0, 0, fb_width, fb_height)
```

in both `set_2d()` and `set_3d()`.

The projection calculations remain based on `width` and `height`.
