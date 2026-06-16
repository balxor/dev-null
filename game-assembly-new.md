# GameAssembly.dll ROX: Versi Lama vs Baru

June 2026 - Himura.

ROX (Ragnarok X: Next Generation) diatur dalam Unity IL2CPP. Semua logika game dikompilasi menjadi kode native di `GameAssembly.dll`. Update 16 Juni 2026 terdapat beberapa perubahan struktur mendasar pada proteksi game.

## Perbedaan

| Properti | Versi Lama (Sep 2025) | Versi Baru (Jun 2026) |
|----------|----------------------|----------------------|
| Ukuran file | 75,628,544 bytes | 71,987,712 bytes |
| MD5 original | `563DF57DF275...` | `30DC4B4FCAFD...` |
| IL2CPP version | 24.4 | 24.4 |
| GSDK version | 3.23.0.0 | 3.23.0.0 |
| GP version | 2.3.1 | 2.3.1 |

File mengecil jadi 3.6 MB. Metadata `global-metadata.dat` juga berubah: dari 19.3 MB menjadi 18.9 MB dan lokasinya pindah dari `RO_SEAGame\il2cpp_data\` ke `rox_Data\il2cpp_data\`.

## Perbedaan Proteksi

### Versi Lama: In-Process GP

ByteDance Game Protect (GP) ditanam langsung ke dalam process game. `GameAssembly.dll` punya 7 P/Invoke bridge function - melakukan panggilan dari managed code ke native `gpHackerProc.dll`. Masing-masing bridge adalah instruksi `call` di alamat:

```
0x1609380, 0x16093F0, 0x16094A0, 0x1609510,
0x1F19500, 0x1F19590, 0x1F19640
```

**Cara kerja versi lama:**

1. GameAssembly.dll dimuat -> bridge functions dipanggil saat inisialisasi
2. Bridge memanggil `gpHackerProc.dll` yang ada di dalam process yang sama
3. gpHackerProc membuka IPC shared memory (`GP_PACKER_B`, `27516shareP_CHILD_PROC`)
4. gpHackerProc menjalankan integrity check periodik via `CreateThreadpoolTimer`
5. gpHackerProc mengirim heartbeat ke server ByteDance via `OpenSSL 3.1.2` + `libcurl`
6. GP SDK (`gp.dll`, `gsdk.dll`) menangani device registration, config cloud dan reporting

Untuk menonaktifkan proteksi versi lama, 7 bridge function di-NOP - setiap instruksi `call` diganti `0x90`. Tanpa bridge, gpHackerProc tetap jalan tapi tidak pernah dipanggil dari GameAssembly.

### Versi Baru: Watchdog Process + 5 Managed Protection

Arsitektur proteksi dirombak total. GP tidak lagi menanam kode ke dalam process game. gpHackerProc sekarang jalan di **process terpisah** yang berfungsi sebagai watchdog.

```
Launcher ROX.exe
├── Watchdog (rox.exe, PID 20484)
│   ├── gpHackerProc.dll  ← monitor dari luar
│   └── gpShell.dll
│
└── Game utama (rox.exe, PID 11156)
    ├── gp.dll (6.98 MB)  ← Game Protect SDK
    ├── gpm.dll            ← GP Monitor
    ├── gpmperf.dll        ← GP Performance
    ├── gsdk.dll (v3.23.0) ← ByteDance GSDK
    ├── gumiho_platform.dll ← anti-fraud
    ├── parfait.dll        ← crash/error monitor
    └── GameAssembly.dll   ← managed protection layer
```

**Lima managed protection di GameAssembly.dll:**

#### 1. CheatManager (TypeDefIndex 13846)
```
class CheatManager {
    void ForceKickOff()              // kick dan banned player
    bool SkipCheatCheck()            // cek apakah cheat check bisa berfungsi
    bool SubmitCheat(string, int, int, int)  // laporkan cheat ke server
}
```
Setiap kali anomali terdeteksi (speed abnormal, memory tamper), `SubmitCheat` dipanggil dengan parameter attrId (jenis anomali), correctionValue (nilai normal), cheatValue (nilai curang) dan cheatCount (frekuensi). Jika threshold terlampaui, `ForceKickOff` memutus koneksi.

#### 2. GameProtectManager (TypeDefIndex 13899)
```
class GameProtectManager {
    void InitProtectHost(string host)           // inisialisasi host-based protection
    void SetRoleInfo(string, string, string, string, string)  // bind data karakter
    void TencentHostLogin(string roleID)        // login via Tencent host
    string GetDeviceFingerprint()               // ambil finger print device
    void ResetAllHosts()                        // reset semua koneksi host
}
```
Melacak environment lokasi game berjalan: NetEase host, Tencent host, atau standalone. `GetDeviceFingerprint` mengumpulkan data hardware/OS untuk identifikasi device.

#### 3. NetSecProtect v1.0.1 (TypeDefIndex 12283)
```
static class NetSecProtect {
    void Init(string appKey)
    void initSDK(string appKey)
    string exportIoctl(RequestCmdID request)     // komunikasi IOCTL
    string getDataSign(string data, int alg)     // tanda tangan data
    string htpIoctl(RequestCmdID, string data)   // HTP IOCTL
    void setRoleId/Info(...)
    string getInfo()
    string EncodeLocal/DecodeLocal(...)
}
```
Network security layer dengan enkripsi lokal dan pipe-based IPC. Command ID-nya mencakup emulator detection (`Cmd_GetEmulatorName = 1`), root detection (`Cmd_IsRootDevice = 2`) dan device ID (`Cmd_DeviceID = 3`). Pertukaran data via named pipe (`Cmd_OpenPipe = 4`, `Cmd_RecvPipe = 5`, `Cmd_ClosePipe = 6`).

#### 4. NetHeartBeat (TypeDefIndex 12850)
```
class NetHeartBeat {
    void registInfoReceiver(InfoReceiver)
    void recvDataThread()         // thread penerima data
    string openPipe()             // buka pipe heartbeat
    string recvPipe()             // baca pipe
    string closePipe()            // tutup pipe
}
```
NetEase heartbeat monitor. Thread terpisah membaca named pipe untuk mendeteksi apakah process game masih jalan dan tidak dimodifikasi. Jika pipe putus atau data tidak valid, trigger alert.

#### 5. ByteDanceSDKManager (TypeDefIndex 13915)
```
class ByteDanceSDKManager : PlatformManager {
    override void InitSDK()        // inisialisasi ByteDance SDK
    void InitTGRPService()         // TGRP downloader service
    void InitWebID()               // WebID registration
    override void InitFoxStark()   // FoxStark mini-program SDK
    void GMSDKInit()               // GMSDK v2024.3.8.0
}
```
Pusat integrasi SDK ByteDance. Menginisialisasi Gumiho (anti-fraud), FoxStark (mini-program/webview), TGRP (downloader), WebID (device registration) dan GMSDK (game middleware). Field `isEmulator` melacak apakah game berjalan di emulator.

## Mekanisme Keseluruhan

**Alur deteksi versi baru:**

1. Watchdog (PID 20484) spawn game utama (PID 11156) dan memonitor dari luar via `gpHackerProc.dll`
2. Game utama memuat `gp.dll` dan `gsdk.dll` saat startup - ini tetap native, tidak bisa di-unload
3. `ByteDanceSDKManager.InitSDK()` dipanggil -> inisialisasi semua sub-SDK
4. `GameProtectManager.InitProtectHost()` -> bind ke host provider (NetEase/Tencent)
5. `NetSecProtect.Init()` -> setup enkripsi lokal dan pipe detection
6. `NetHeartBeat` -> mulai heartbeat thread via named pipe
7. `CheatManager` -> standby, siap menerima laporan anomali dari berbagai subsystem
8. Jika anomali terdeteksi -> `SubmitCheat()` -> threshold -> `ForceKickOff()`

**Perbedaan penting dari versi yang lama:**

| Aspek | Versi Lama | Versi Baru |
|-------|-----------|------------|
| Lokasi gpHackerProc | In-process | Watchdog process |
| P/Invoke bridges di GameAssembly | 7 fungsi `call` | Tidak ada |
| Managed protection | Tidak ada | 5 lapis (27 fungsi) |
| Emulator detection | Tidak ada | `Cmd_GetEmulatorName` |
| Lua verification | Tidak ada | `SignatureLoader` RSA+SHA1 |
| Heartbeat | IPC shared memory | Named pipe thread |

## Menonaktifkan Proteksi Versi Baru

Karena tidak ada P/Invoke bridge yang perlu di-NOP, strateginya berbeda. Tiap fungsi protection di-patch langsung di entry point-nya dengan instruksi `RET`.

**Teknik patching:**

Tiga byte pattern yang dapat digunakan:
- `C3` - RET (1 byte). Untuk fungsi void.
- `33 C0 C3` - XOR EAX,EAX; RET (3 byte). Untuk fungsi yang mengembalikan integer/pointer/boolean false.
- `B0 01 C3` - MOV AL,1; RET (3 byte). Untuk fungsi boolean yang harus mengembalikan true.

Patch diterapkan di file offset (bukan RVA), karena file yang dimodifikasi di dalam disk.

**Daftar patch:**

| Kelas | Fungsi | Offset | Pattern |
|-------|--------|--------|---------|
| CheatManager | ForceKickOff | 0xB02C90 | C3 |
| CheatManager | SkipCheatCheck | 0xB02E50 | B0 01 C3 |
| CheatManager | SubmitCheat | 0xB02EC0 | B0 01 C3 |
| GameProtectManager | InitProtectHost | 0x11553F0 | C3 |
| GameProtectManager | SetRoleInfo | 0x1155500 | C3 |
| GameProtectManager | TencentHostLogin | 0x11555E0 | C3 |
| GameProtectManager | GetDeviceFingerprint | 0x1155360 | 33 C0 C3 |
| GameProtectManager | ResetAllHosts | 0x11554B0 | C3 |
| NetSecProtect | Init | 0xD8E070 | C3 |
| NetSecProtect | initSDK | 0xD8E550 | C3 |
| NetSecProtect | exportIoctl | 0xD8E280 | 33 C0 C3 |
| NetSecProtect | getDataSign | 0xD8E440 | 33 C0 C3 |
| NetSecProtect | htpIoctl | 0xD8E4F0 | 33 C0 C3 |
| NetSecProtect | setRoleId | 0xD8E630 | C3 |
| NetSecProtect | setRole | 0xD8E700 | C3 |
| NetSecProtect | setRoleInfo | 0xD8E680 | C3 |
| NetSecProtect | getInfo | 0xD8E4A0 | 33 C0 C3 |
| NetHeartBeat | registInfoReceiver | 0xD8AD40 | C3 |
| NetHeartBeat | recvDataThread | 0xD8A8F0 | C3 |
| NetHeartBeat | openPipe | 0xD8A880 | 33 C0 C3 |
| NetHeartBeat | recvPipe | 0xD8ACD0 | 33 C0 C3 |
| NetHeartBeat | closePipe | 0xD8A5C0 | 33 C0 C3 |
| ByteDanceSDKManager | InitSDK | 0xDE73A0 | C3 |
| ByteDanceSDKManager | InitTGRPService | 0xDE7440 | C3 |
| ByteDanceSDKManager | InitWebID | 0xDE7620 | C3 |
| ByteDanceSDKManager | InitFoxStark | 0xDE7180 | C3 |
| ByteDanceSDKManager | GMSDKInit | 0xDE5D20 | C3 |

27 fungsi, 5 kelas, 1 file target.

**Efek setelah patch:**

Saat fungsi dipanggil, CPU langsung menemui `RET` di byte pertama. Stack frame tidak terbentuk, body fungsi tidak dieksekusi. Fungsi yang mengembalikan value langsung mengembalikan 0/null/false/true sesuai pattern. Semua subsystem proteksi menjadi inert.

Native DLL (`gp.dll`, `gsdk.dll`, `gumiho_platform.dll`) tetap di-load oleh OS karena ada di import table process - tapi tidak ada managed code yang memanggilnya. Watchdog process (`gpHackerProc.dll`) tetap berjalan di process terpisah, namun tidak menerima data atau alert dari game utama karena semua reporter sudah mati.

**Menjalankan patcher:**

```powershell
# Tutup game terlebih dahulu
powershell -ExecutionPolicy Bypass -File .\patch-silence.ps1
```

Backup otomatis dibuat sebagai `GameAssembly.dll.bak.silence`. Untuk restore:

```powershell
Copy-Item GameAssembly.dll.bak.silence GameAssembly.dll -Force
```

**Verifikasi:**

Setelah patch, jalankan game. Jalankan cheat script/tools. Jika 27 hook di dalam `rox-silence.js` tidak diperlukan lagi (game tetap jalan tanpa crash), patch berhasil.

## Batasan

Patch ini menonaktifkan managed-code protection di GameAssembly.dll. Native DLL tidak dimodifikasi karena:

- `gp.dll` dan `gsdk.dll` adalah komponen GSDK core - game crash jika di-unload
- `gpHackerProc.dll` di watchdog process - process terpisah, tidak bisa diakses dari patcher tanpa kill process
- `parfait.dll` untuk crash reporting - tidak mengganggu Frida injection
- `xlua.dll` untuk Lua hotfix - diperlukan game untuk berfungsi

Native DLL tetap berjalan tetapi tidak menerima trigger dari managed code, sehingga secara fungsional inert.

## Test Result

Game dijalankan setelah patch tanpa Frida - tidak crash. Dengan Frida + attack-hack script - 157 hits/35 detik, 0.6% reject rate, tidak ada kick atau disconnect. 27 fungsi proteksi confirmed inert.

## Note

Saya tidak bisa membagi script rox-silence.js di dalam repository ini.
/dev/null
