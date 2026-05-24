# Click-Through Region Feature

This is a fork of the `window_manager` package that adds support for **rectangular click-through regions** on Windows. Instead of making the entire window click-through, you can now specify a single rectangular area that passes mouse events to windows below it, while the rest of your application remains fully interactive.

## Overview

The original `setIgnoreMouseEvents()` method makes the **entire window** transparent to mouse events. This modified version allows you to define a rectangular region that is click-through while keeping the rest of the window interactive.

### Use Cases

- Create "hole" effects that allow interaction with windows behind your app
- Build transparent overlay windows with interactive UI surrounding the hole
- Implement custom hit-test regions for special window behaviors
- Create developer tools or magnifying glass overlays

## API Changes

### Dart API (lib/src/window_manager.dart)

```dart
Future<void> setIgnoreMouseEvents(
  bool ignore, {
  bool forward = false,
  double? x,
  double? y,
  double? width,
  double? height,
  double? devicePixelRatio,
}) async
```

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `ignore` | `bool` | Enable or disable click-through behavior |
| `forward` | `bool` | (Legacy, kept for compatibility) |
| `x` | `double?` | X coordinate of the click-through rectangle (logical pixels, relative to window top-left) |
| `y` | `double?` | Y coordinate of the click-through rectangle (logical pixels, relative to window top-left) |
| `width` | `double?` | Width of the click-through rectangle (logical pixels) |
| `height` | `double?` | Height of the click-through rectangle (logical pixels) |
| `devicePixelRatio` | `double?` | Device pixel ratio for coordinate conversion |

### Behavior

- **When `ignore = true` and rectangle provided**: Only the specified rectangular region passes mouse events to windows below; the rest of the window is interactive.
- **When `ignore = true` and no rectangle**: The entire window becomes click-through (original behavior).
- **When `ignore = false`**: Normal mouse event handling resumes.

## Implementation Details

### Windows Backend (windows/window_manager.cpp)

The implementation uses Windows **Region APIs** to create a window region that excludes the click-through area:

1. **Region Creation**: A full window region is created using `CreateRectRgn()`
2. **Region Subtraction**: The click-through rectangle is subtracted from the region using `CombineRgn()` with `RGN_DIFF`
3. **Region Application**: The modified region is applied to the window using `SetWindowRgn()`
4. **Window Refresh**: When the window moves or resizes, `UpdateClickThroughRegion()` recalculates and reapplies the region

#### Key Changes

**Class Members (lines 75-80):**
```cpp
RECT click_through_rect_ = {0, 0, 0, 0};
double click_through_x_ = 0;
double click_through_y_ = 0;
double click_through_width_ = 0;
double click_through_height_ = 0;
bool is_ignore_mouse_events_ = false;
```

**SetIgnoreMouseEvents() Method (lines 1090-1126):**
- Extracts coordinates and device pixel ratio from Dart
- Stores the rectangle dimensions with device pixel ratio scaling
- Calls `UpdateClickThroughRegion()` to apply the region

**UpdateClickThroughRegion() Method (lines 1164-1183):**
- Recalculates absolute coordinates based on current window position
- Creates a full window region and subtracts the click-through rectangle
- Applies the region using `SetWindowRgn()`

**WM_EXITSIZEMOVE Handler (line 224):**
- Updates the region after window move/resize to maintain correct hit-test area

## Usage Examples

### Example 1: Simple Click-Through Rectangle

```dart
import 'package:window_manager/window_manager.dart';

// Enable click-through only in the rectangle at (100, 100) with size (200, 200)
await windowManager.setIgnoreMouseEvents(
  true,
  x: 100,
  y: 100,
  width: 200,
  height: 200,
  devicePixelRatio: window.devicePixelRatio,
);

// Disable click-through (normal behavior)
await windowManager.setIgnoreMouseEvents(false);
```

### Example 2: Dynamic Click-Through Region

```dart
import 'package:flutter/material.dart';
import 'package:window_manager/window_manager.dart';

class MagnifyingGlassOverlay extends StatefulWidget {
  @override
  State<MagnifyingGlassOverlay> createState() => _MagnifyingGlassOverlayState();
}

class _MagnifyingGlassOverlayState extends State<MagnifyingGlassOverlay> {
  Offset _holePosition = Offset(0, 0);
  double _holeSize = 150;

  @override
  void initState() {
    super.initState();
    _updateClickThroughRegion();
  }

  Future<void> _updateClickThroughRegion() async {
    await windowManager.setIgnoreMouseEvents(
      true,
      x: _holePosition.dx,
      y: _holePosition.dy,
      width: _holeSize,
      height: _holeSize,
      devicePixelRatio: MediaQuery.of(context).devicePixelRatio,
    );
  }

  @override
  Widget build(BuildContext context) {
    return MouseRegion(
      onHover: (event) async {
        setState(() {
          _holePosition = event.position;
        });
        await _updateClickThroughRegion();
      },
      child: Container(
        color: Colors.black54,
        child: Center(
          child: Container(
            width: _holeSize,
            height: _holeSize,
            decoration: BoxDecoration(
              border: Border.all(color: Colors.white, width: 2),
              borderRadius: BorderRadius.circular(_holeSize / 2),
            ),
          ),
        ),
      ),
    );
  }
}
```

### Example 3: Entire Window Click-Through (Original Behavior)

```dart
// Make entire window click-through (don't provide rectangle coordinates)
await windowManager.setIgnoreMouseEvents(true);

// Resume normal behavior
await windowManager.setIgnoreMouseEvents(false);
```

## Coordinate System

- **X, Y coordinates**: Relative to the window's top-left corner in logical pixels
- **Width, Height**: Size of the rectangle in logical pixels
- **Device Pixel Ratio**: Provided by Flutter and used to convert logical pixels to physical pixels for Windows API

### Example Calculation

```dart
final dpr = MediaQuery.of(context).devicePixelRatio;

// Logical coordinates (what you define in Flutter)
double logicalX = 100;
double logicalY = 100;
double logicalWidth = 200;
double logicalHeight = 200;

// Physical coordinates (Windows uses these)
// int physicalX = 100 * dpr;
// int physicalY = 100 * dpr;
// (calculated internally by the plugin)

await windowManager.setIgnoreMouseEvents(
  true,
  x: logicalX,
  y: logicalY,
  width: logicalWidth,
  height: logicalHeight,
  devicePixelRatio: dpr,
);
```

## Platform Support

- ✅ **Windows**: Fully supported with rectangle precision
- ❌ **macOS**: Not yet implemented (uses legacy behavior)
- ❌ **Linux**: Not yet implemented (uses legacy behavior)

## Known Limitations

1. **Rectangular regions only**: Complex shapes are not supported; only axis-aligned rectangles
2. **Windows only**: macOS and Linux platforms retain the original all-or-nothing behavior
3. **Single region**: Only one click-through rectangle per window (extends the implementation if multiple regions needed)
4. **Coordinate scaling**: Must provide accurate `devicePixelRatio` for correct hit-testing

## Performance Considerations

- Region creation/update is typically fast (single operation per update)
- Recommended to debounce frequent updates if tracking mouse/cursor position
- No significant CPU overhead once the region is applied

## Debugging

To verify the click-through region is working:

1. Create a Flutter window with `setIgnoreMouseEvents(true, x: ..., y: ..., width: ..., height: ...)`
2. Move another window partially behind the hole and verify you can interact with it
3. Verify UI elements outside the region still respond to clicks normally

## Implementation Details for Developers

### Windows Region API Reference

The implementation relies on these Windows APIs:

- `CreateRectRgn()`: Create a rectangular region
- `CombineRgn()`: Combine regions (we use `RGN_DIFF` to subtract)
- `DeleteObject()`: Clean up region handles
- `SetWindowRgn()`: Apply the region to the window
- `GetWindowRect()`: Get current window coordinates

### Thread Safety

The implementation is not thread-safe. All calls to `setIgnoreMouseEvents()` should be made from the main/UI thread.

### Rebuilding on Window Resize

The `UpdateClickThroughRegion()` method is automatically called on `WM_EXITSIZEMOVE` to recalculate the region with the new window position. This ensures the hit-test area remains at the correct screen coordinates even if the window moves or resizes.
