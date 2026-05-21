![Flayme Logo](https://spectrion.github.io/assets/IMG_4118.png)

> **Hybrid open-source kernel — natively runs Linux apps, Windows EXE/PE, Android APKs, and .jyd packages.**

________________________________________________________________________________________________________________________________________________________________________
**Version:** 1.0.0 "Ember"  
**Base:** Linux 6.x  
**Architecture:** Hybrid kernel (monolithic core + modular runtime subsystems)  
**License:** GPL-2.0 (kernel), LGPL-2.1 (wine-flayme), Apache-2.0 (android-rt/FART)

---

## What This Is

Flayme is a **kernel** (not a full OS) that extends the Linux kernel with:

| Feature | How |
|---|---|
| **Run Linux ELF apps** | Native — zero overhead, standard binfmt_elf |
| **Run Windows .EXE/.PE** | Wine-Flayme — a modified Wine fork (LGPL) with FRB backend |
| **Run Android .APK** | FART — Flayme Android Runtime (AOSP ART port, Apache-2.0) |
| **Run .jyd packages** | Jayde — Flayme's native universal app format |
| **Cross-runtime IPC** | FRB (Flayme Runtime Bus) — `/dev/frb` kernel driver |

**None of this is CPU emulation.** All binaries run native x86-64 or ARM64 code on the CPU. Only API/syscall translation happens.

---

## Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                         USER SPACE                               │
│                                                                  │
│  Linux ELF      Windows PE        Android APK      .jyd          │
│     │               │                 │              │           │
│  (native)      Wine-Flayme          FART          Jayde          │
│   binfmt_elf   (Wine fork)       (ART port)      jyd-runner      │
│     │          ntdll.so            art.so            │           │
│     │          Win32→Linux       DEX→native      (unpacks +      │
│     │          syscall shim      JIT/AOT          re-executes)   │
│     └──────────────┴─────────────────┴──────────────┘           │
│                              │                                   │
│                    /dev/frb  │  (Flayme Runtime Bus)             │
├──────────────────────────────▼───────────────────────────────────┤
│                      FLAYME KERNEL                               │
│                                                                  │
│  binfmt_pe  binfmt_apk  binfmt_jyd  frb.ko  (kernel modules)    │
│                                                                  │
│  Standard Linux kernel subsystems:                               │
│  scheduler | memory | VFS | network | drivers                    │
└──────────────────────────────────────────────────────────────────┘
                         HARDWARE
```

## Kernel Subsystems

### `fs/binfmt/binfmt_pe.c`
Kernel module. Detects `MZ` magic on `execve()`, hands off to Wine-Flayme's `wine64` loader. Wine translates Win32 API calls → Linux syscalls. **Not an emulator** — Windows x86-64 code runs natively on the CPU.

### `fs/binfmt/binfmt_apk.c`
Kernel module. Detects ZIP+`AndroidManifest.xml`, hands off to FART. FART runs `dex2oat` to AOT-compile DEX bytecode → native x86-64/ARM64 machine code, then boots ART. **Not an emulator** — compiled app code runs natively.

### `fs/binfmt/binfmt_jyd.c`
Kernel module. Detects `JAYD` magic, hands off to `jyd-runner`. Runner selects best payload (ELF > PE > APK) and exec's it through the appropriate handler above.

### `kernel/frb/frb.c` — Flayme Runtime Bus
Character device at `/dev/frb`. Provides a lockless ring-buffer IPC channel between the three runtime subsystems. Used for:
- Sharing GPU surfaces between apps (unified compositor)
- Passing file descriptors across runtime boundaries
- Android Intents system-wide (any runtime can send/receive)
- Clipboard sync between Windows, Android, and Linux apps
- Wine's `wineserver` replacement (`frb-winserver`)

### `wine-flayme/` — Wine Fork
A fork of [Wine](https://www.winehq.org/) (LGPL-2.1) with:
- **No X11** — uses Flayme's Wayland compositor via FRB directly
- **FRB backend** replaces wineserver Unix socket protocol
- **ntdll.so** translates `NtCreateFile`, `NtAllocateVirtualMemory`, etc. → Linux syscalls
- **DXVK built-in** — DirectX 9/10/11/12 → Vulkan translation
- Windows path mapping: `C:\Users\` → `/home/`, etc.

### `android-rt/` — FART (Flayme Android Runtime)
AOSP ART (Apache-2.0) ported to run on Flayme/Linux:
- `dex2oat` AOT-compiles DEX bytecode → native code
- Binder IPC bridged over FRB (`FRB_MSG_INTENT`, etc.)
- SurfaceFlinger replaced by Wayland client
- Audio via PipeWire (not Android HAL)
- Camera/sensors via Linux V4L2/IIO

### `tools/jayde/jayde.c` — Jayde
The `.jyd` package management tool. Can create, inspect, sign, install, and run `.jyd` files. Also converts existing ELF/PE/APK apps into `.jyd` format.

---

## .jyd Package Format

`.jyd` (pronounced "jade") is Flayme's native universal app container:

```
[256-byte header "JAYD"] [manifest.json] [sections...] [signature]
```

Each section is zstd-compressed and SHA-256 verified. A `.jyd` can contain:
- A Linux ELF binary (preferred — fastest)
- A Windows PE binary (fallback)
- An Android APK (fallback)
- Any combination ("universal apps")

The Jayde tool handles everything:
```bash
# Convert an existing app
jayde pack --apk myapp.apk --exe myapp.exe --elf myapp -o myapp.jyd

# Run it
jayde run myapp.jyd

# Or just exec it directly (kernel handles it)
./myapp.jyd
```

---

## Building

```bash
# Install dependencies
make deps

# Download Linux 6.x source, apply Flayme patches, configure
make defconfig

# Build everything
make all

# Or step by step:
make kernel        # kernel image
make modules       # binfmt_pe, binfmt_apk, binfmt_jyd, frb
make wine-flayme   # Win32 compatibility layer
make fart          # Android runtime
make jayde         # .jyd tool

# Test in QEMU
make run-qemu

# Build bootable ISO
make logo && make iso
```

## Contributing

Contributions welcome! Key areas:
- `kernel/frb/` — FRB performance improvements
- `wine-flayme/` — Win32 API coverage (patches against Wine)
- `android-rt/` — Android framework API coverage
- `fs/binfmt/` — additional binary format handlers
- `tools/jayde/` — Jayde GUI frontend

See `Documentation/contributing.md`.

---

## Why "Not an emulator"?

| Term | Meaning |
|---|---|
| **Emulator** | Simulates a *different CPU* in software (e.g. running ARM code on x86) — **slow** |
| **Compatibility layer** | Translates *system calls / API calls* — the actual machine code runs on the real CPU — **fast** |

Flayme is a compatibility layer. Wine, WSL1, and Flayme all use the same principle. The application's compiled machine code runs directly on your CPU at full speed. Only the OS-level calls (file I/O, memory allocation, window creation) are intercepted and translated.

---

*Flayme Kernel — because your kernel shouldn't limit what you can run.*
