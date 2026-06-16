# GameAssembly.dll: Pre-Update vs Post-Update

Comparison between the original GameAssembly.dll (ROX SEA v4.1.3, pre-June 2026) and the updated version (June 16, 2026).

## File Properties

| Property | Pre-Update (Original) | Post-Update (June 2026) |
|----------|----------------------|-------------------------|
| File size | 75,628,544 bytes (72.1 MB) | 71,987,712 bytes (68.7 MB) |
| MD5 | `563DF57DF275AE2645C9601E65AA5514` | `30DC4B4FCAFDB697FCCB72DB8C8C6382` |
| MD5 (patched) | `8A126E4E36C547444C0B96E7D618C7BA` | Not required |
| IL2CPP version | 24.4 (dumped as 24.5) | 24.4 (dumped as 24.5) |
| Metadata size | 19,298,352 bytes | 18,935,988 bytes |
| GSDK version | 3.23.0 | 3.23.0 |
| GP version | 2.3.1 | 2.3.1 |

## Anti-Cheat: ByteDance GP

### Pre-Update

The original DLL contained 7 P/Invoke bridge functions that called into ByteDance GP anti-cheat libraries (`gp.dll`, `gpHackerProc.dll`, `gsdk.dll`). These bridges were located at file offsets:

```
0x1609380, 0x16093F0, 0x16094A0, 0x1609510,
0x1F19500, 0x1F19590, 0x1F19640
```

Each bridge was a `call` instruction (`E8 xx xx xx xx`) targeting a function inside `gpHackerProc.dll`. Without NOP patching, the bridges would:

1. Initialize GP runtime monitoring
2. Register IPC shared memory channels (`GP_PACKER_B`, `27516shareP_CHILD_PROC`)
3. Start periodic integrity checks via `CreateThreadpoolTimer`
4. Send heartbeat/report data to ByteDance servers

The patched version replaced all 7 `call` instructions with `NOP` (0x90), permanently disabling GP initialization.

### Post-Update

The new DLL does not contain these P/Invoke bridges. Evidence:

- File is 3.6 MB smaller (removed code/data)
- `DllImport` references to `gp.dll`, `gpHackerProc.dll`, and `gsdk.dll` not found in `dump.cs`
- Frida hook scripts run without crash on the unpatched DLL
- The game process does not spawn a second `rox.exe` instance (gpHackerProc)

ByteDance GP anti-cheat was removed from this build.

## RVA Changes

All function addresses shifted. The IL2CPP metadata and code layout were restructured.

| Function | Old RVA | New RVA | Delta |
|----------|---------|---------|-------|
| `NetworkManager.Send(MessageId, IMessage)` | `0x727A70` | `0xA9A7E0` | +0x372D70 |
| `NetworkManager.OnReceive` | `0x727130` | `0xA9A130` | +0x373000 |
| `GetPublicCDEndTime` | `0x90BD30` | `0xCB7460` | +0x3AB730 |
| `set_NormalSkillCDEndTime` | `0x911FF0` | `0xCBD720` | +0x3AB730 |
| `ShowCD` | `0x910330` | `0xCAE120` | +0x39DDF0 |
| `OnCastSkill` | `0x90DB10` | `0xBC3600` | +0x2B5AF0 |
| `SkillConfigConditionsEnough` | `0x7F1390` | `0xABAE60` | +0x2C9AD0 |
| `SkillFailed` | `0x910810` | `0xCBBF40` | +0x3AB730 |
| `GetAnimationSpeed` | `0x916020` | `0xCC17A0` | +0x3AB780 |
| `GetRealAnimationLast` | `0x9161A0` | `0xCC1920` | +0x3AB780 |
| `StartSkillAnimation` | `0x7B7BD0` | `0xD63AB0` | +0x5ABEE0 |
| `SendEncodeMsgWithResponse` | `0x727480` | `0xA9A1F0` | +0x372D70 |

## Protobuf Message Structures

IL2CPP object field offsets (CastSkillCommand, CastSkillNotify, etc.) remain unchanged from the pre-update version. The update restructured native code layout but did not modify protobuf message schemas.

## New Bundles

The update downloaded 642 new UnityFS bundle files. `BundleList.txt` was updated. New `global-metadata.dat` at `rox_Data\il2cpp_data\Metadata\` (previously the metadata was at `RO_SEAGame\il2cpp_data\Metadata\`).

## ByteDance Custom Header Removed

The pre-update `global-metadata.dat` had an 8-byte ByteDance custom prefix before the standard IL2CPP magic bytes (`AF 1B B1 FA`). The post-update metadata starts directly with the IL2CPP magic bytes. The custom header was ByteDance-specific obfuscation, removed in this build.

## Migration Checklist

When updating from pre-June 2026 to post-June 2026:

```
1. Replace GameAssembly.dll (auto-updated by launcher)
2. Re-run Il2CppDumper on new DLL + new metadata
3. Update all Frida script RVAs from the mapping above
4. NOP patching no longer required (GP removed)
5. Verify hooks install without crash
6. Test attack speed / CD bypass functionality
```
