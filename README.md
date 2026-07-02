# Hermes Desktop on Windows: Blank Screen, Renderer Crash, and Lost Settings Fix

This README documents a real Windows troubleshooting session for the Hermes Agent Desktop app. It covers:

- Hermes not opening
- Hermes opening to a blank black screen
- Renderer crash loops
- Start Menu shortcut fixes
- Zoom and window position not persisting
- Local patches that made the app usable

The machine in this case was Windows, using the Hermes desktop build installed under:

```text
C:\Users\<you>\AppData\Local\hermes\hermes-agent
```

The desktop executable was:

```text
C:\Users\<you>\AppData\Local\hermes\hermes-agent\apps\desktop\release\win-unpacked\Hermes.exe
```

Replace `<you>` with your Windows username.

## Short Version

The app was failing because the Electron/Chromium renderer process was crashing on Windows.

The reliable launch fix was to start Hermes Desktop with:

```text
--no-sandbox
```

The Start Menu shortcut needed to point to the real desktop executable:

```text
C:\Users\<you>\AppData\Local\hermes\hermes-agent\apps\desktop\release\win-unpacked\Hermes.exe
```

And the shortcut arguments needed to be:

```text
--no-sandbox
```

`--disable-gpu` by itself was not enough.

## Symptoms Observed

### 1. Hermes did not open

Clicking Hermes from the Start Menu appeared to do nothing.

Windows Event Viewer showed repeated app crashes:

```text
Faulting application name: Hermes.exe
Exception code: 0x80000003
Faulting application path:
C:\Users\<you>\AppData\Local\hermes\hermes-agent\apps\desktop\release\win-unpacked\Hermes.exe
```

### 2. Running `Hermes.exe --disable-gpu` in PowerShell showed CLI help

PowerShell resolved `Hermes.exe` to the Hermes CLI, not the desktop app:

```text
C:\Users\<you>\AppData\Local\hermes\hermes-agent\venv\Scripts\hermes.exe
```

That produced:

```text
hermes: error: unrecognized arguments: --disable-gpu
```

This was not the desktop executable. The desktop executable is the one under:

```text
apps\desktop\release\win-unpacked\Hermes.exe
```

### 3. Hermes opened but showed a blank black screen

After the initial crash was worked around, the window appeared but the UI area was blank.

The Hermes desktop log showed renderer crashes:

```text
[hermes] [renderer] render-process-gone reason=crashed exitCode=-2147483645
[hermes] [renderer] suppressing reload: 3 crashes within 60000ms (likely a crash loop)
```

### 4. Zoom and window settings reset

Window position and zoom settings appeared to reset randomly.

This happened because:

- Window position was saved in `window-state.json`
- Zoom was saved only in Chromium Local Storage
- During crash recovery, Electron cache/storage folders were moved or recreated
- That wiped or bypassed the zoom value
- If Hermes crashed before a clean move/resize/close cycle, `window-state.json` might not get written

## Root Causes

### Root Cause 1: Wrong executable being launched manually

PowerShell `Hermes.exe` resolved to the CLI executable:

```text
C:\Users\<you>\AppData\Local\hermes\hermes-agent\venv\Scripts\hermes.exe
```

The desktop app is:

```text
C:\Users\<you>\AppData\Local\hermes\hermes-agent\apps\desktop\release\win-unpacked\Hermes.exe
```

### Root Cause 2: Electron/Chromium GPU startup crash

The app logged:

```text
GPU process isn't usable. Goodbye.
```

This points to a Chromium/Electron graphics startup problem on this Windows setup.

### Root Cause 3: Electron sandbox renderer crash

Even after GPU-related flags, the renderer continued to crash. The working fix was:

```text
--no-sandbox
```

With `--no-sandbox`, Hermes loaded its UI and backend successfully:

```text
[hermes] [boot] Hermes backend is ready. Finalizing desktop startup
```

### Root Cause 4: Local updates overwrote the shortcut

The Start Menu shortcut was later overwritten back to:

```text
--disable-gpu
```

That made the blank-screen issue return.

The correct argument was:

```text
--no-sandbox
```

### Root Cause 5: Zoom persistence used fragile storage

Hermes persisted zoom using renderer Local Storage:

```js
const ZOOM_STORAGE_KEY = 'hermes:desktop:zoomLevel'
```

That lives under Electron/Chromium app storage:

```text
C:\Users\<you>\AppData\Roaming\Hermes\Local Storage\leveldb
```

If this storage is cleared, moved, corrupted, or recreated during renderer crash recovery, zoom resets.

## Quick User Fix

### Fix the Start Menu shortcut

Run this in PowerShell:

```powershell
$exe = "$env:LOCALAPPDATA\hermes\hermes-agent\apps\desktop\release\win-unpacked\Hermes.exe"
$lnk = Join-Path $env:APPDATA "Microsoft\Windows\Start Menu\Programs\Hermes.lnk"

$shell = New-Object -ComObject WScript.Shell
$shortcut = $shell.CreateShortcut($lnk)
$shortcut.TargetPath = $exe
$shortcut.Arguments = "--no-sandbox"
$shortcut.WorkingDirectory = Split-Path $exe

$icon = Join-Path (Split-Path $exe) "resources\icon.ico"
if (Test-Path $icon) {
  $shortcut.IconLocation = "$icon,0"
} else {
  $shortcut.IconLocation = "$exe,0"
}

$shortcut.Description = "Hermes Agent Desktop"
$shortcut.Save()
```

Then launch Hermes from the Start Menu.

## One-Shot Repair Script

This script:

- Stops any currently running Hermes desktop processes
- Fixes the Start Menu shortcut
- Adds the required `--no-sandbox` argument
- Moves stale Electron graphics/cache folders aside
- Launches Hermes
- Prints process/window status
- Prints the latest desktop log lines

```powershell
$ErrorActionPreference = "SilentlyContinue"

$exe = Join-Path $env:LOCALAPPDATA "hermes\hermes-agent\apps\desktop\release\win-unpacked\Hermes.exe"
$lnk = Join-Path $env:APPDATA "Microsoft\Windows\Start Menu\Programs\Hermes.lnk"
$cacheRoot = Join-Path $env:APPDATA "Hermes"
$desktopLog = Join-Path $env:LOCALAPPDATA "hermes\logs\desktop.log"

if (-not (Test-Path -LiteralPath $exe)) {
  Write-Error "Hermes desktop executable was not found: $exe"
  exit 1
}

Get-Process |
  Where-Object { $_.Path -eq $exe -or $_.ProcessName -eq "Hermes" } |
  Stop-Process -Force

$shell = New-Object -ComObject WScript.Shell
$shortcut = $shell.CreateShortcut($lnk)
$shortcut.TargetPath = $exe
$shortcut.Arguments = "--no-sandbox"
$shortcut.WorkingDirectory = Split-Path $exe

$icon = Join-Path (Split-Path $exe) "resources\icon.ico"
if (Test-Path -LiteralPath $icon) {
  $shortcut.IconLocation = "$icon,0"
} else {
  $shortcut.IconLocation = "$exe,0"
}

$shortcut.Description = "Hermes Agent Desktop"
$shortcut.Save()

$stamp = Get-Date -Format yyyyMMdd-HHmmss
foreach ($name in @(
  "GPUCache",
  "DawnGraphiteCache",
  "DawnWebGPUCache",
  "ShaderCache",
  "Code Cache",
  "Session Storage"
)) {
  $path = Join-Path $cacheRoot $name
  if (Test-Path -LiteralPath $path) {
    Move-Item -LiteralPath $path -Destination "$path.old-$stamp" -Force
  }
}

Start-Process -FilePath $lnk
Start-Sleep -Seconds 18

$running = @(Get-Process | Where-Object { $_.Path -eq $exe })
$windows = @($running | Where-Object { $_.MainWindowHandle -ne 0 })

[pscustomobject]@{
  Shortcut = $lnk
  Target = $shortcut.TargetPath
  Arguments = $shortcut.Arguments
  RunningCount = $running.Count
  WindowCount = $windows.Count
  WindowTitles = ($windows.MainWindowTitle -join " | ")
  Pids = ($running.Id -join ",")
} | Format-List

if (Test-Path -LiteralPath $desktopLog) {
  "Latest Hermes desktop log:"
  Get-Content -LiteralPath $desktopLog -Tail 30
}
```

### Verify the shortcut

```powershell
$lnk = Join-Path $env:APPDATA "Microsoft\Windows\Start Menu\Programs\Hermes.lnk"
$shell = New-Object -ComObject WScript.Shell
$s = $shell.CreateShortcut($lnk)

[pscustomobject]@{
  Target = $s.TargetPath
  Arguments = $s.Arguments
  WorkingDirectory = $s.WorkingDirectory
}
```

Expected:

```text
Target:    C:\Users\<you>\AppData\Local\hermes\hermes-agent\apps\desktop\release\win-unpacked\Hermes.exe
Arguments: --no-sandbox
```

## Diagnostic Script

Use this before and after applying the fix. It checks:

- Which executable the Start Menu shortcut points to
- Which arguments are configured
- Whether Hermes is currently running
- Whether a visible Hermes window exists
- Recent Hermes crash entries from Windows Event Log
- Recent Hermes desktop log lines

```powershell
$exe = Join-Path $env:LOCALAPPDATA "hermes\hermes-agent\apps\desktop\release\win-unpacked\Hermes.exe"
$lnk = Join-Path $env:APPDATA "Microsoft\Windows\Start Menu\Programs\Hermes.lnk"
$desktopLog = Join-Path $env:LOCALAPPDATA "hermes\logs\desktop.log"

"== Shortcut =="
if (Test-Path -LiteralPath $lnk) {
  $shell = New-Object -ComObject WScript.Shell
  $s = $shell.CreateShortcut($lnk)
  [pscustomobject]@{
    Shortcut = $lnk
    Target = $s.TargetPath
    Arguments = $s.Arguments
    WorkingDirectory = $s.WorkingDirectory
  } | Format-List
} else {
  "Start Menu shortcut not found: $lnk"
}

"== Running Processes =="
Get-Process |
  Where-Object { $_.Path -eq $exe -or $_.ProcessName -eq "Hermes" } |
  Select-Object ProcessName,Id,MainWindowTitle,MainWindowHandle,Responding,StartTime,Path |
  Format-List

"== Recent Windows Crash Entries =="
Get-EventLog -LogName Application -Newest 20 -ErrorAction SilentlyContinue |
  Where-Object {
    $_.Source -match "Application Error|Windows Error Reporting" -and
    $_.Message -match "Hermes.exe"
  } |
  Select-Object TimeGenerated,Source,EntryType,Message |
  Format-List

"== Hermes Desktop Log =="
if (Test-Path -LiteralPath $desktopLog) {
  Get-Content -LiteralPath $desktopLog -Tail 80
} else {
  "No desktop log found: $desktopLog"
}
```

## Manual Launch Test

Use the full desktop executable path:

```powershell
& "$env:LOCALAPPDATA\hermes\hermes-agent\apps\desktop\release\win-unpacked\Hermes.exe" --no-sandbox
```

Do not use only:

```powershell
Hermes.exe --no-sandbox
```

That may call the CLI instead of the desktop app.

## Logs to Check

### Hermes desktop log

```text
C:\Users\<you>\AppData\Local\hermes\logs\desktop.log
```

Useful command:

```powershell
Get-Content "$env:LOCALAPPDATA\hermes\logs\desktop.log" -Tail 80
```

Bad signs:

```text
[renderer] render-process-gone reason=crashed exitCode=-2147483645
GPU process isn't usable. Goodbye.
```

Good sign:

```text
[boot] Hermes backend is ready. Finalizing desktop startup
```

### Windows Event Viewer via PowerShell

```powershell
Get-EventLog -LogName Application -Newest 20 |
  Where-Object { $_.Message -match "Hermes.exe" } |
  Select-Object TimeGenerated,Source,EntryType,Message |
  Format-List
```

## Cache Cleanup Used During Recovery

These folders were moved aside during debugging to force Electron/Chromium to regenerate graphics/cache state:

```text
C:\Users\<you>\AppData\Roaming\Hermes\GPUCache
C:\Users\<you>\AppData\Roaming\Hermes\DawnGraphiteCache
C:\Users\<you>\AppData\Roaming\Hermes\DawnWebGPUCache
C:\Users\<you>\AppData\Roaming\Hermes\ShaderCache
C:\Users\<you>\AppData\Roaming\Hermes\Code Cache
C:\Users\<you>\AppData\Roaming\Hermes\Session Storage
```

PowerShell:

```powershell
$cacheRoot = "$env:APPDATA\Hermes"
$stamp = Get-Date -Format yyyyMMdd-HHmmss

foreach ($name in @(
  "GPUCache",
  "DawnGraphiteCache",
  "DawnWebGPUCache",
  "ShaderCache",
  "Code Cache",
  "Session Storage"
)) {
  $path = Join-Path $cacheRoot $name
  if (Test-Path -LiteralPath $path) {
    Move-Item -LiteralPath $path -Destination "$path.old-$stamp" -Force
  }
}
```

Warning: clearing or moving Electron storage can reset some UI state, especially if the app stores state in Local Storage.

## Local Code Patches Applied

These patches were applied locally under:

```text
C:\Users\<you>\AppData\Local\hermes\hermes-agent\apps\desktop\electron\main.cjs
```

### Patch 1: Disable GPU-related Chromium paths on Windows

The app already had logic to disable hardware acceleration for remote displays. The local patch made Windows use the safer path too and added stronger Chromium switches.

Conceptual patch:

```js
if (REMOTE_DISPLAY_REASON || IS_WINDOWS) {
  app.disableHardwareAcceleration()
  app.commandLine.appendSwitch('disable-gpu')
  app.commandLine.appendSwitch('disable-gpu-sandbox')
  app.commandLine.appendSwitch('disable-gpu-rasterization')
  app.commandLine.appendSwitch('disable-accelerated-2d-canvas')
  app.commandLine.appendSwitch('disable-accelerated-video-decode')
  app.commandLine.appendSwitch('disable-direct-composition')
  app.commandLine.appendSwitch('disable-webgl')
  app.commandLine.appendSwitch('disable-webgl2')
  app.commandLine.appendSwitch(
    'disable-features',
    'Vulkan,VizDisplayCompositor,UseSkiaRenderer,CanvasOopRasterization,DirectComposition'
  )
  app.commandLine.appendSwitch('disable-gpu-compositing')
}
```

However, this was not enough by itself. The renderer still required:

```text
--no-sandbox
```

### Patch 2: Persist zoom in a JSON file

Original Hermes behavior persisted zoom in renderer Local Storage:

```js
const ZOOM_STORAGE_KEY = 'hermes:desktop:zoomLevel'
```

The local patch added:

```js
const DESKTOP_ZOOM_STATE_PATH = path.join(app.getPath('userData'), 'zoom-state.json')
```

And helper functions:

```js
function readZoomState() {
  try {
    const raw = JSON.parse(fs.readFileSync(DESKTOP_ZOOM_STATE_PATH, 'utf8'))
    return clampZoomLevel(Number(raw?.zoomLevel))
  } catch {
    return null
  }
}

function writeZoomState(zoomLevel) {
  try {
    fs.mkdirSync(path.dirname(DESKTOP_ZOOM_STATE_PATH), { recursive: true })
    fs.writeFileSync(
      DESKTOP_ZOOM_STATE_PATH,
      JSON.stringify({ zoomLevel: clampZoomLevel(zoomLevel) }, null, 2)
    )
  } catch (error) {
    rememberLog(`[zoom] json persist failed: ${error?.message || error}`)
  }
}
```

Then `setAndPersistZoomLevel` was changed to write JSON too:

```js
function setAndPersistZoomLevel(window, zoomLevel) {
  if (!window || window.isDestroyed()) return
  const next = clampZoomLevel(zoomLevel)
  window.webContents.setZoomLevel(next)
  writeZoomState(next)

  window.webContents
    .executeJavaScript(
      `try { localStorage.setItem(${JSON.stringify(ZOOM_STORAGE_KEY)}, ${JSON.stringify(String(next))}) } catch {}`
    )
    .catch(error => rememberLog(`[zoom] persist failed: ${error?.message || error}`))
}
```

And `restorePersistedZoomLevel` was changed to prefer the JSON file:

```js
function restorePersistedZoomLevel(window) {
  if (!window || window.isDestroyed()) return

  const saved = readZoomState()
  if (saved != null) {
    window.webContents.setZoomLevel(saved)
    return
  }

  window.webContents
    .executeJavaScript(
      `(() => { try { return localStorage.getItem(${JSON.stringify(ZOOM_STORAGE_KEY)}) } catch { return null } })()`
    )
    .then(stored => {
      if (stored == null || !window || window.isDestroyed()) return
      const level = clampZoomLevel(Number(stored))
      window.webContents.setZoomLevel(level)
    })
    .catch(error => rememberLog(`[zoom] restore failed: ${error?.message || error}`))
}
```

After this patch, zoom is saved here:

```text
C:\Users\<you>\AppData\Roaming\Hermes\zoom-state.json
```

### Patch 3: Save window state as soon as the window first opens

Hermes persisted window geometry on resize, move, maximize, unmaximize, and close. A local patch also persisted once when the window first became visible.

Conceptual patch:

```js
mainWindow.once('ready-to-show', () => {
  if (mainWindow && !mainWindow.isDestroyed()) {
    mainWindow.show()
    schedulePersistWindowState()
  }
})
```

This ensures:

```text
C:\Users\<you>\AppData\Roaming\Hermes\window-state.json
```

is created earlier instead of waiting for a clean close or manual resize.

## Rebuilding Hermes Desktop After Patching

After editing source files, rebuild the desktop package:

```powershell
hermes desktop --force-build --build-only
```

Then re-apply the Start Menu shortcut argument:

```text
--no-sandbox
```

Important: the rebuild or a Hermes update may overwrite the Start Menu shortcut.

## Verify Hermes Is Running

```powershell
$exe = "$env:LOCALAPPDATA\hermes\hermes-agent\apps\desktop\release\win-unpacked\Hermes.exe"

Get-Process |
  Where-Object { $_.Path -eq $exe } |
  Select-Object ProcessName,Id,MainWindowTitle,MainWindowHandle,Responding,StartTime
```

Good output includes:

```text
MainWindowTitle: Hermes
Responding: True
```

## Verify State Files

```powershell
Get-Item "$env:APPDATA\Hermes\window-state.json","$env:APPDATA\Hermes\zoom-state.json" -ErrorAction SilentlyContinue
```

Window state example:

```json
{
  "x": 243,
  "y": 56,
  "width": 1221,
  "height": 800,
  "isMaximized": false
}
```

Zoom state example:

```json
{
  "zoomLevel": 0.2
}
```

`zoom-state.json` appears after changing zoom once with:

```text
Ctrl + +
Ctrl + -
Ctrl + 0
```

## What To Tell Users

If Hermes opens to a blank screen on Windows:

1. Check whether the shortcut points to the real desktop executable.
2. Add `--no-sandbox` to the shortcut arguments.
3. Clear only graphics/cache folders if needed.
4. Check `desktop.log` for renderer crashes.

The most important fix from this case was:

```text
Hermes.exe --no-sandbox
```

using the full desktop executable path.

## Caveats

### `--no-sandbox` is a workaround

Disabling Chromium/Electron sandboxing reduces renderer isolation. It should be treated as a compatibility workaround, not an ideal long-term fix.

For a proper upstream fix, Hermes should investigate why Electron 40.10.2 renderer sandboxing crashes on this Windows setup.

### Hermes updates may overwrite local fixes

Observed behavior:

- The Start Menu shortcut was overwritten from `--no-sandbox` back to `--disable-gpu`
- Local source patches can be overwritten by updates or rebuilds
- Rebuilding the app can require re-applying the shortcut fix

If the blank screen comes back, check the shortcut arguments first.

## Suggested Upstream Improvements

For Hermes maintainers:

1. Add a Windows-safe launch fallback when repeated renderer crashes are detected.
2. Persist zoom in main-process JSON instead of only renderer Local Storage.
3. Preserve Start Menu launch arguments during updates.
4. Consider exposing a setting or environment variable for `--no-sandbox`.
5. Log the exact launch arguments and userData path on desktop startup.
6. Add diagnostics for:
   - GPU process crash
   - renderer sandbox crash
   - missing/overwritten shortcut args
   - failed zoom/window-state persistence

## Summary

The issue was not simply “Hermes does not open.”

There were multiple layers:

1. PowerShell was calling the CLI `Hermes.exe`, not the desktop `Hermes.exe`.
2. The desktop app crashed due to Chromium/Electron GPU startup.
3. After that was partially fixed, the renderer still crashed due to sandboxing.
4. The correct runtime workaround was `--no-sandbox`.
5. Hermes updates overwrote that shortcut argument.
6. Zoom reset because it lived in Electron Local Storage, which was fragile during crash recovery.
7. A local patch moved zoom persistence into `zoom-state.json`.

The practical fix for most users:

```text
Use the real desktop executable and launch it with --no-sandbox.
```
