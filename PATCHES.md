# HAZE Cyberdeck OS - Firmware Patches v0.6.2

## Overview

This document describes the 5 critical safety fixes applied to TEmbedOS v0.6.1.

---

## Patch 1: Preferences Initialization Validation

**File**: `TEmbedOS.ino`  
**Functions Affected**: `loadSettings()`, `saveSettings()`  
**Lines**: ~375-420 (depends on original file)

### Problem
The firmware called `Preferences.begin()` but ignored its return value. If initialization failed (corrupted flash, full storage), settings would never load or save, but the firmware would not report the error.

### Solution

#### Step 1: Add global flag
After `static Preferences hazePreferences;`, add:

```cpp
static bool preferencesReady = false;
```

#### Step 2: Update `loadSettings()`
Replace the entire function with the version that checks the return value:

```cpp
static void loadSettings() {
  defaultsSettings();

  preferencesReady = hazePreferences.begin("haze", false);

  if (!preferencesReady) {
    Serial.println("[SETTINGS] Preferences initialization failed");
    applyTheme();
    return;
  }

  // ... existing code continues
}
```

#### Step 3: Update `saveSettings()`
Replace with:

```cpp
static void saveSettings() {
  if (!preferencesReady) {
    Serial.println("[SETTINGS] Save skipped: Preferences unavailable");
    return;
  }

  hazePreferences.putUChar("rotation", settings.rotation);
  // ... rest of save code
}
```

### Result
- Settings failures are now visible in Serial output
- Prevents silent data loss
- Safe fallback to defaults if NVS corrupted

---

## Patch 2: BLE Callback and Initialization Safety

**File**: `TEmbedOS.ino`  
**Functions Affected**: `HazeWriteCallbacks::onWrite()`, `initializeBle()`  
**Severity**: HIGH — potential null-pointer dereference

### Problem
If BLE initialization fails or a callback is invoked with a null characteristic, the firmware would crash or exhibit undefined behavior.

### Solution

#### Step 1: Guard BLE callback
In `HazeWriteCallbacks::onWrite()`, add null check at the start:

```cpp
void onWrite(BLECharacteristic* characteristic) override {
  if (characteristic == nullptr) {
    Serial.println("[BLE] Null characteristic");
    return;
  }

  std::string value =
      characteristic->getValue();

  if (value.empty()) {
    return;
  }

  // ... existing command processing
}
```

#### Step 2: Guard BLE server creation
In `initializeBle()`, after `BLEDevice::init()`, add checks:

```cpp
#if HAZE_BLE_AVAILABLE
  if (bleRunning) return;

  BLEDevice::init("HAZE CYBERDECK");

  bleServer = BLEDevice::createServer();
  if (bleServer == nullptr) {
    Serial.println("[BLE] Failed to create server");
    bleRunning = false;
    return;
  }

  bleServer->setCallbacks(new HazeServerCallbacks());

  BLEService* service =
      bleServer->createService(HAZE_SERVICE_UUID);

  if (service == nullptr) {
    Serial.println("[BLE] Failed to create service");
    bleRunning = false;
    return;
  }

  bleWriteCharacteristic =
      service->createCharacteristic(
          HAZE_WRITE_UUID,
          BLECharacteristic::PROPERTY_WRITE |
          BLECharacteristic::PROPERTY_WRITE_NR);

  bleNotifyCharacteristic =
      service->createCharacteristic(
          HAZE_NOTIFY_UUID,
          BLECharacteristic::PROPERTY_NOTIFY |
          BLECharacteristic::PROPERTY_READ);

  if (bleWriteCharacteristic == nullptr ||
      bleNotifyCharacteristic == nullptr) {
    Serial.println("[BLE] Failed to create characteristics");
    bleRunning = false;
    return;
  }

  // ... rest of initialization
#endif
```

### Result
- BLE failures are logged clearly
- Firmware continues safely if BLE unavailable
- No crashes from null dereferencing

---

## Patch 3: Display Write Transactions

**File**: `TEmbedOS.ino`  
**Function Affected**: `redrawScheduler()`  
**Lines**: Partial redraw section (status bar and face)

### Problem
Partial redraws (status bar, face animation) did not use `startWrite()`/`endWrite()` transactions. This can cause:
- Graphical artifacts (old pixels not erased)
- Inconsistent SPI state
- Race conditions if display updates overlap

### Solution

Replace the partial redraw section:

```cpp
// OLD CODE (problematic):
if (statusBarDirty && !idleMode) {
  drawStatusBar();
  statusBarDirty = false;
}

if (faceDirty &&
    currentScreen == SCREEN_HOME &&
    !idleMode) {
  drawHome();
  drawStatusBar();
  faceDirty = false;
}

// NEW CODE (safe):
if (statusBarDirty && !idleMode) {
  displayDevice.startWrite();
  drawStatusBar();
  displayDevice.endWrite();
  statusBarDirty = false;
}

if (faceDirty &&
    currentScreen == SCREEN_HOME &&
    !idleMode) {
  displayDevice.startWrite();
  displayDevice.fillScreen(activeTheme.background);
  drawInsetBorder();
  drawHome();
  drawStatusBar();
  displayDevice.endWrite();
  faceDirty = false;
}
```

### Result
- Face animation no longer leaves stale pixels
- Consistent SPI transaction handling
- Smooth visual updates

---

## Patch 4: WiFi Scan Cleanup

**File**: `TEmbedOS.ino`  
**Function Affected**: `handleScreenInput()`, SCREEN_WIFI case  
**Lines**: WiFi selection == 2 (scan action)

### Problem
The WiFi scan did not call `WiFi.scanDelete()` after getting results. This leaves scan data in memory, potentially corrupting subsequent scans or wasting heap.

### Solution

In the WiFi scan handler, after `WiFi.scanNetworks()`:

```cpp
if (wifiSelection == 2) {
  wifiState = WIFI_SCANNING;
  contentDirty = true;

  int result = WiFi.scanNetworks(false, true);

  Serial.printf("[WIFI] Scan result=%d\n", result);

  WiFi.scanDelete();  // <-- ADD THIS LINE

  wifiState = WIFI_IDLE;
  contentDirty = true;
}
```

### Result
- Memory leak prevented
- Repeated scans work reliably
- Cleaner heap usage

---

## Patch 5: Setup Initialization Order (Optional)

**File**: `TEmbedOS.ino`  
**Function Affected**: `setup()`  
**Severity**: LOW (improves startup clarity, not critical)

### Problem
Settings were loaded *after* the display initialized. This meant:
- Display test used defaults, then orientation changed
- Startup felt unpredictable
- Theme could mismatch on first boot

### Solution

Reorder initialization:

```cpp
// OLD:
defaultsSettings();
applyTheme();
if (!initializeDisplay()) { ... }
loadSettings();
applyOrientation();

// NEW:
defaultsSettings();
loadSettings();      // <-- MOVE UP
applyTheme();
if (!initializeDisplay()) { ... }
applyOrientation();
```

### Result
- Saved settings applied before first draw
- More predictable startup behavior
- Clearer code flow

---

## Patch 6: Remote Control Feedback (Optional)

**File**: `TEmbedOS.ino`  
**Function Affected**: `handleScreenInput()`, SCREEN_REMOTE case  
**Severity**: LOW (UX clarity only)

### Problem
The remote screen printed the URL but did not actually send a command. The UI implied action was taken when it wasn't.

### Solution

In the remote select handler, change:

```cpp
// OLD:
Serial.printf("[REMOTE] %s\n", REMOTE_BASE_URL);

// NEW:
Serial.printf("[REMOTE] Authorized local target selected: %s\n",
              REMOTE_BASE_URL);
// No network request is sent in this firmware revision.
```

Optionally, show an error to the user:

```cpp
showErrorScreen(ERROR_REMOTE,
                "REMOTE NOT YET IMPLEMENTED");
```

### Result
- Clearer expectations
- No false impression of functionality

---

## Compilation Checklist

After applying all patches:

1. ✅ All `#include` statements present
2. ✅ No undefined references to `bleServer`, `bleWriteCharacteristic`, `bleNotifyCharacteristic`
3. ✅ `preferencesReady` flag declared and used
4. ✅ `displayDevice.startWrite()` and `.endWrite()` properly paired
5. ✅ `WiFi.scanDelete()` called after scan
6. ✅ `setup()` initialization order: defaults → load → theme → display → orient

---

## Testing Checklist

On hardware:

- [ ] Display initializes to correct orientation
- [ ] Settings persist after power cycle
- [ ] Face animation smooth on home screen (no ghosting)
- [ ] WiFi scan completes and doesn't corrupt heap
- [ ] BLE advertises or fails gracefully
- [ ] Serial output shows all init messages (no corruption)
- [ ] Safe boot mode accessible (hold BOOT on reset)
- [ ] Idle mode activates after 45 seconds

---

## Version History

| Version | Date | Changes |
|---------|------|----------|
| 0.6.1 | Original | Base firmware |
| 0.6.2 | 2025-09-08 | Patches 1-6 applied |

---

## Support

If issues occur after patching:

1. Check **Serial output** for error messages
2. Verify **pin definitions** match your T-Embed variant
3. Test **display offset** (35, 0 vs. 0, 0) if image is clipped
4. Confirm **ESP32 core version** (2.0.17 or later recommended)
5. Try **safe boot mode**: hold BOOT button during reset
