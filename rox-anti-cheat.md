# ROX Anti-Cheat & Protection Analysis

Post-update analysis of Ragnarok X: Next Generation (PC version). Game version 4.1.3, updated June 16, 2026.  
by: Himura, DSTH.  

## Architecture

The old GP architecture injected `gpHackerProc.dll` directly into the main game process via P/Invoke bridges in `GameAssembly.dll`. These bridges were NOP-patched to disable GP.

The new architecture separates protection into a **watchdog process** that monitors the main game from outside.

```
Launcher: ROX.exe (PID 41824)
├── ROX.exe (PID 29900) - GPU process
├── ROX.exe (PID 39648) - Utility/network
├── ROX.exe (PID 16836) - Renderer
├── ROX.exe (PID 6084)  - Renderer
├── ROX.exe (PID 8672)  - Crashpad handler
│
├── WATCHDOG: rox.exe (PID 20484) [4 threads, 9MB]
│   └── gpHackerProc.dll + gpShell.dll loaded
│
└── MAIN GAME: rox.exe (PID 11156) [217 threads, 3.6GB]
    ├── gp.dll, gpm.dll, gpmperf.dll, gsdk.dll loaded
    ├── UnityCrashHandler64.exe
    ├── cefsubprocess.exe (x4)
    └── parfait_crash_handler.exe
```

## Loaded Protection DLLs (Main Game)

| DLL | Path | Size | Purpose |
|-----|------|------|---------|
| `gp.dll` | `rox_Data\Plugins\x86_64\gp.dll` | 6.98 MB | ByteDance Game Protect SDK |
| `gpm.dll` | `rox_Data\Plugins\x86_64\gpm.dll` | 289 KB | GP Monitor |
| `gpmperf.dll` | `rox_Data\Plugins\x86_64\gpmperf.dll` | 241 KB | GP Performance Monitor |
| `gsdk.dll` | `rox_Data\Plugins\x86_64\gsdk.dll` | 4.0 MB | ByteDance GSDK v3.23.0 |
| `gpShell.dll` | `RO_SEAGame\gpShell.dll` | 396 KB | GP Shell (loaded in both processes) |
| `gumiho_platform.dll` | `rox_Data\Plugins\x86_64\` | 4.6 MB | ByteDance anti-fraud platform |
| `parfait.dll` | `rox_Data\Plugins\x86_64\parfait.dll` | 1.0 MB | APM crash/error monitoring |
| `deviceregister_shared.dll` | `rox_Data\Plugins\x86_64\` | 270 KB | Device registration |
| `logsdk.dll` | `rox_Data\Plugins\x86_64\logsdk.dll` | 2.3 MB | Logging SDK |
| `tgrpdownloader.dll` | `rox_Data\Plugins\x86_64\` | 2.5 MB | TGRP downloader |
| `sscronet.dll` | `rox_Data\Plugins\x86_64\` | 7.2 MB | Cronet network stack |
| `ssgamesdkcronet.dll` | `rox_Data\Plugins\x86_64\` | 1.7 MB | Game SDK Cronet |
| `xlua.dll` | `rox_Data\Plugins\x86_64\` | 726 KB | XLua engine |
| `libcrypto-1_1-x64.dll` | `rox_Data\Plugins\x86_64\` | 3.4 MB | OpenSSL 1.1 |

## Watchdog Process (PID 20484)

Separate `rox.exe` instance loading:

- `gpHackerProc.dll` - GP hacker/anti-tamper module (3.5 MB)
- `gpShell.dll` - GP shell interface (396 KB)

4 threads, 9 MB RAM. Lightweight external monitor. Parent of the main game process.

## Managed Protection Classes (IL2CPP)

### CheatManager
- Namespace: `Dream` - TypeDefIndex: 13846
- `ForceKickOff()` - forcibly disconnect the player
- `SkipCheatCheck()` - bypass check (debug/dev only)
- `SubmitCheat(attrId, correctionValue, cheatValue, cheatCount)` - report detected cheat

### GameProtectManager
- Namespace: `Dream` - TypeDefIndex: 13899
- `IsNetEaseHost`, `IsTencentHost` - host-based protection flags
- `InitProtectHost()` - initialize host-specific protection
- `SetRoleInfo()` - bind role data for tracking
- `GetDeviceFingerprint()` - device fingerprinting
- `TencentHostLogin()` - Tencent-specific login validation
- `ResetAllHosts()` - reset protection state

### NetSecProtect v1.0.1
- Namespace: global - TypeDefIndex: 12283
- Network security layer with IPC pipe communication
- `Init(appKey)` - initialize with application key
- `initSDK()` - SDK initialization
- `exportIoctl()` / `htpIoctl()` - IOCTL-based communication
- `EncodeLocal()` / `DecodeLocal()` - local encryption
- `getDataSign()` - data signing
- `getInfo()` / `setRoleInfo()` - info exchange
- `registInfoReceiver()` - register callback

Request command IDs:
```
Cmd_GetEmulatorName = 1  - Emulator detection
Cmd_IsRootDevice    = 2  - Root detection
Cmd_DeviceID        = 3  - Device ID
Cmd_OpenPipe        = 4  - IPC pipe open
Cmd_RecvPipe        = 5  - IPC pipe receive
Cmd_ClosePipe       = 6  - IPC pipe close
Cmd_GetHTPVersion   = 7  - HTP version
```

### NetHeartBeat
- Namespace: `NetEase` - TypeDefIndex: 12850
- Threaded heartbeat monitor using named pipes
- `openPipe()`, `recvPipe()`, `closePipe()`, `recvDataThread()`

### SignatureLoader
- Namespace: `XLua` - TypeDefIndex: 15236
- Lua script integrity verification - RSA + SHA1
- `load_and_verify(ref filepath)` - verify Lua file before execution

### ByteDanceSDKManager
- Namespace: `Dream` - TypeDefIndex: 13915
- Central ByteDance SDK manager
- `isEmulator` - emulator detection flag
- `isNineTaleInit` - Gumiho platform init flag
- `RuGameAdvancedInjection injection` - Lua-to-Unity bridge injection
- `InitSDK()`, `InitFoxStark()`, `InitWebID()`, `InitTGRPService()`

### GumihoBridge
- Links to `gumiho_platform.dll` (ByteDance anti-fraud - "Nine-Tailed Fox")
- `GumihoCallNativeSync()` / `GumihoCallNativeASync()` - native calls
- `GumihoRegisterListener()` - event listener registration

### Additional Classes

| Class | Purpose |
|-------|---------|
| `NetHTProtect` | Network/HTP protection layer (TypeDefIndex 7838) |
| `SignatureHelper` | Signature utilities (TypeDefIndex 666) |
| `HashUtility` | Hash calculation (TypeDefIndex 7842, 10100) |
| `FoxStark` namespace | ByteDance mini-program/webview SDK |
| `MainSDK` (GMSDK v2024.3.8.0) | `SdkGetDeviceId()`, `SdkGetInstallId()`, `SdkGetSDKVersion()` |
| `AccountBytedanceService` | Real-name auth, anti-addiction |
| `InfoAndErrorReporter` | Error/telemetry batch upload |
| `DreamGameProtectManagerWrap` | XLua wrapper exposing protection to Lua |

## config.xml - Protection Skeleton

```xml
<Configuration>
    <FileVerification forceVerify="0">
        <!-- Disabled - no files listed -->
    </FileVerification>
    <inject>
        <!-- Disabled - DLL injection config commented out -->
    </inject>
    <SignatureBanList>
        <!-- Disabled - no signature bans -->
    </SignatureBanList>
</Configuration>
```

Three protection mechanisms exist in skeleton form. Currently `forceVerify="0"` and all entries are commented out. These can be activated by server-side config push via GSDK cloud control.

## Unloaded Protection DLLs (Present on Disk)

| DLL | Size | Purpose |
|-----|------|---------|
| `uwa.dll` | 162 KB | Unity Web Analyzer |
| `dream.dll` | 1.9 MB | Dream engine module |
| `rnu.dll` | 1.8 MB | Rainbow Native UI |
| `ruyoga.dll` | 99 KB | RU Yoga layout |
| `TRAE.dll` | 3.1 MB | Tencent audio engine |
| `UDT.dll` | 190 KB | UDT data transport |
| `gmesdk.dll` | 6.9 MB | GMESDK middleware |
| `sdkencryptedappticket64.dll` | 1.0 MB | Steam ticket |
| `steam_api64.dll` | 295 KB | Steam API |
| `ByteRTCCWrapper.dll` | 51 KB | ByteDance RTC |
| `VolcEngineRTCAudio.dll` | 29.1 MB | Volcano Engine audio |
| `xPlatform.dll` | 506 KB | Cross-platform helper |

## Old Plugin Directory (Deprecated)

`C:\RO_SEA\RO_SEAGame\Plugins\x86_64\` contains old versions, no longer loaded:

- `gpm.dll` - 287 KB (old version)
- `gpmperf.dll` - 240 KB (old version)

## Migration Summary

| Aspect | Pre-Update | Post-Update |
|--------|-----------|-------------|
| GP architecture | In-process P/Invoke bridges | Watchdog process + loaded DLLs |
| NOP patch needed | Yes (7 bridges) | No |
| `gpHackerProc.dll` | In main process | In watchdog process |
| `gp.dll` location | `Plugins\x86_64\` | `rox_Data\Plugins\x86_64\` |
| Protection layers | 1 (GP only) | 9+ (GP + CheatManager + NetSecProtect + etc.) |
| Frida compatibility | Works after NOP patch | Works without patch |
| Detection risk | Low (GP bridges NOP'd) | Low (watchdog is external, doesn't hook into GameAssembly) |

## Last but /dev/null

ByteDance GP was not removed. It was restructured into a multi-layer protection system with a separate watchdog process. The protection is now **more sophisticated** but **less invasive** to the main game process. This architectural change removes the need for GameAssembly.dll patching and makes Frida-based instrumentation more stable.
