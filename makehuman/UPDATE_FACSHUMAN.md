# FACSHuman & MakeHuman Update Report (April 2026)

This document summarizes the debugging and modifications performed to ensure compatibility with modern environments (Python 3.11+, NumPy 2.x) and to resolve graphical rendering issues.

## 1. Core Engine Compatibility (NumPy 2.x & Python 3)

### NumPy 2.x "Binary Mode" Fixes
NumPy 2.x removed the binary mode for `np.fromstring`. All occurrences were migrated to `np.frombuffer` or `np.array(list())` depending on the context.
- **Modified**: `lib/image_qt.py`, `core/files3d.py`, `makehuman.py`, `shared/proxy.py`.
- **String Handling**: Updated `toNumpyString` and `fromNumpyString` in `makehuman.py` to explicitly encode/decode UTF-8 bytes to prevent crashes with non-ASCII characters.

### Deprecated Method Migration
Replaced all occurrences of the deprecated/removed `.tostring()` method with `.tobytes()` for NumPy arrays.
- **Modified**: `lib/image.py`, `lib/image_qt.py`, `core/files3d.py`, `makehuman.py`, `shared/proxy.py`.

## 2. Graphical & Rendering Improvements

### Texture Stability & Persistence
Fixed "scrambled" or flickering textures caused by temporary memory reclamation.
- **QImage Data Ownership**: Added `.copy()` to `QImage` initializations in `lib/image.py` and `lib/image_qt.py`. This ensures the UI/Renderer owns the pixel data rather than holding a view into a temporary NumPy buffer.
- **Memory Safety**: Ensured NumPy arrays created from `im.bits()` in `lib/image_qt.py` are copied to avoid use-after-free glitches.

### OpenGL Data Contiguity & Memory Safety
Resolved "glitched" geometry and shader artifacts by enforcing C-contiguous memory layout, while maintaining memory safety for pointers.
- **A() Helper Fix**: Rewrote the `A()` utility function in `lib/glmodule.py` to be a robust recursive flattener. 
    *   **The Issue**: Previously, it relied on `np.array(arg).flatten()`, which in NumPy 2.x created object arrays for custom objects like `Color`, leading to `ValueError: setting an array element with a sequence` when setting light properties.
    *   **The Fix**: Explicitly flattens iterables and NumPy arrays into a standard list before final array creation.
- **Vertex Attributes**: Ensured vertices, normals, UVs, colors, and tangents are C-contiguous in `module3d.py`. 
- **Pointer Safety**: Reverted `np.ascontiguousarray()` from `glVertexPointer` and other pointer functions in `lib/glmodule.py`. 
    *   **The Issue**: Passing temporary arrays to these functions caused segfaults (and the "Python is likely shutting down" error) because the memory was freed before drawing.
    *   **The Fix**: Used persistent, already-contiguous arrays from the objects.
- **Shader Uniforms**: Maintained `np.ascontiguousarray()` in `lib/shader.py` for uniforms (safe as they are copied immediately by the driver).
- **Texture Uploads**: Maintained contiguity enforcement in `lib/texture.py` for safe `glTexImage2D` calls.

### Robust Off-screen Rendering (Take one shot)
Fixed the issue where the model disappeared from the main view after taking a screenshot.
- **Try...Finally Blocks**: Wrapped `renderToBuffer` and `renderAlphaMask` in `try...finally` to ensure OpenGL state (framebuffers, blending, clear color) and window dimensions are always restored, even if an error occurs.
- **Camera State Management**:
    *   Invalidate all cameras *before* rendering a shot to ensure the correct aspect ratio is used.
    *   Invalidate all cameras *after* rendering a shot to force a recalculation for the main screen dimensions. This prevents the model from being projected incorrectly (the "disappearing" bug).

### Depth & Culling Fixes
- **High-Precision Depth**: Upgraded the render buffer from 16-bit to 24-bit (`GL_DEPTH_COMPONENT24`) in `lib/glmodule.py` to eliminate Z-fighting (flickering surfaces).
- **Transparency & Angles**: Disabled backface culling for `Teeth`, `Tongue`, and `Eyes` in `shared/proxy.py` to prevent these parts from disappearing when viewed from specific angles or through transparent materials.

## 3. FACSHuman Plugin Suite Patches

### General Plugin Compatibility
All FACS plugins were updated to use the Python 3 shebang and robust file handling.
- **Files**: `7_FACSHuman.py`, `8_FACSAnim.py`, `9_FACS_scene_editor.py`.
- **Encoding**: Switched to `with open(..., encoding='utf-8')` for all JSON operations to prevent encoding errors.

### 7_FACSHuman.py
- **Subprocess Fixes**: Resolved `TypeError` in video/stereo rendering by removing manual byte-encoding of command-line arguments. Corrected string/bytes concatenation for FFmpeg filters.

### 8_FACSAnim.py
- **Iteration Safety**: Updated dictionary loops to use `list(dict.keys())` to ensure compatibility with Python 3's dictionary views.

### 9_FACS_scene_editor.py
- **UI Logic**: Fixed a major bug where `G.app.scene` was not restored in `onHide`, causing application-wide state corruption.
- **File Dialogs**: Fixed tuple unpacking for `getOpenFileName` and `getSaveFileName` to match the modern PyQt5 API.
- **Widget Setup**: Corrected `IconListFileChooser` arguments to ensure the "Scene" chooser displays correctly.

---
**Status**: All systems verified compatible with NumPy 2.4.4 and Python 3.11. Rendering is stabilized for modern GPUs.
