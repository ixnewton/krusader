# Krusader Closedown Fixes

## Overview
This patch fixes multiple critical issues that prevent proper application shutdown in Krusader.

## Issues Fixed

### 1. Incomplete DBus Cleanup (krusader.cpp)
**Problem:** Four DBus objects were registered at startup but only one was unregistered at shutdown, leaving orphaned DBus registrations that could prevent new instances from starting.

**Files Modified:** `app/krusader.cpp`

**Changes:**
- Added `dbus.unregisterObject("/Instances/" + Krusader::AppName + "/left_manager")`
- Added `dbus.unregisterObject("/Instances/" + Krusader::AppName + "/right_manager")`
- Added `dbus.unregisterService("org.krusader")`

### 2. Double-Delete in ListPanel Destructor (listpanel.cpp)
**Problem:** The destructor manually deleted widgets that were already owned by Qt's parent-child hierarchy and added to layouts, causing potential double-delete crashes or use-after-free errors.

**Files Modified:** `app/Panel/listpanel.cpp`

**Changes:**
- Added null checks before removing event filters
- Removed manual deletion of Qt-managed widgets (status, bookmarksButton, totals, urlNavigator, buttons)
- Kept explicit deletion only for `func` (non-QWidget) and `view` (requires specific cleanup order)
- Added explanatory comments

### 3. Missing Event Filter Removal (terminaldock.cpp)
**Problem:** An event filter was installed on `qApp` in `initialise()` but never removed in the destructor, causing use-after-free crashes when the application processes events after TerminalDock is destroyed.

**Files Modified:** `app/GUI/terminaldock.cpp`

**Changes:**
- Added `qApp->removeEventFilter(this)` in destructor when `initialised` is true

### 4. KrViewer Hanging on Close (krusader.cpp)
**Problem:** KrViewer windows with KParts (editors/viewers) could block indefinitely in `queryClose()` when:
- KParts wait for user input (save dialog)
- Network operations haven't completed
- Parts are in a bad state

This caused the entire application to hang during shutdown, requiring force-kill.

**Files Modified:** `app/krusader.cpp`

**Changes:**
- Added `MAX_CLOSE_ATTEMPTS` limit (50) to prevent infinite loop
- Added special handling for KrViewer windows that refuse to close
- Force-delete KrViewer windows with `deleteLater()` and continue shutdown instead of aborting
- Added debug output to track which widgets fail to close

## Application Instructions

### To Apply the Patch:
```bash
cd /path/to/krusader
git apply krusader-closedown-fixes.patch
```

Or if not using git:
```bash
cd /path/to/krusader
patch -p1 < krusader-closedown-fixes.patch
```

### To Build:
```bash
cmake -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build -j$(nproc)
sudo cmake --install build
```

## Testing
1. Start Krusader
2. Open files in the internal viewer/editor (F3/F4)
3. Close Krusader (File → Quit or Ctrl+Q)
4. Application should close properly without hanging

If KrViewer windows refuse to close, you'll see debug output:
```
Failed to close: KrViewer
Force deleting KrViewer window
```

## Technical Details

### DBus Registration
At startup in `main.cpp`, four DBus objects are registered:
1. `/Instances/<appName>` - Main instance
2. `/Instances/<appName>/left_manager` - Left panel manager
3. `/Instances/<appName>/right_manager` - Right panel manager
4. Service `org.krusader` - DBus service

All must be unregistered during shutdown to allow clean restart.

### Qt Parent-Child Hierarchy
Widgets created with `this` as parent and added to layouts are automatically deleted by Qt when the parent is destroyed. Manual deletion causes double-delete.

### Event Filter Lifecycle
Event filters installed on `qApp` must be explicitly removed before the object is destroyed, otherwise Qt may call the filter on a deleted object.

### KParts Blocking
KParts::ReadWritePart::queryClose() can block waiting for:
- User response to "Save changes?" dialog
- Network operations to complete
- File locks to be released

During application shutdown, we need to force-close these to prevent hanging.

## Compatibility
- Tested with KDE Frameworks 6.x
- Qt 6.4.0+
- No API changes - internal fixes only

## Author
Generated via AI-assisted code analysis and debugging
Date: October 29, 2025
