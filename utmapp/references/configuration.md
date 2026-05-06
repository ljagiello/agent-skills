# UTM bundle and configuration reference

Read this when the user wants to hand-edit a `.utm` bundle outside UTM, scripts a custom QEMU argument, or needs to understand the on-disk schema for backup, migration, or templating.

## Contents
- [Bundle layout](#bundle-layout)
- [config.plist top level](#configplist-top-level)
- [QEMU configuration schema](#qemu-configuration-schema)
- [Apple configuration schema](#apple-configuration-schema)
- [Modifying a bundle safely](#modifying-a-bundle-safely)
- [Custom QEMU arguments](#custom-qemu-arguments)
- [Migrating a bundle between hosts](#migrating-a-bundle-between-hosts)

## Bundle layout

```
MyVM.utm/                     ← directory, opaque in Finder ("show package contents")
├── config.plist              ← XML or binary plist; the canonical configuration
├── Data/
│   ├── <uuid>.qcow2          ← virtual disk(s); also seen: .img, .raw
│   ├── efi_vars.fd           ← UEFI NVRAM (QEMU UEFI guests)
│   ├── AuxiliaryStorage      ← Apple-backend NVRAM blob (binary)
│   └── HardwareModel         ← Apple-backend hardware-model blob (binary)
├── Images/                   ← removable-media files (older bundles)
├── view.plist                ← per-host display state (window size, scaling); safe to delete
└── screenshot.png            ← last frame, used as VM tile in UTM's main view
```

A QEMU VM only needs `config.plist` plus its disk images. Apple VMs additionally need `AuxiliaryStorage` and `HardwareModel` (those tie the VM to a specific Apple machine identity — they cannot be regenerated for a macOS guest without a fresh restore).

## config.plist top level

`config.plist` is a property list. Use `plutil` to convert formats:

```bash
plutil -convert xml1 -o - config.plist | less          # human-readable
plutil -convert binary1 config.plist                   # back to binary
plutil -p config.plist                                 # pretty-print
```

The top-level keys you will see in current (`ConfigurationVersion = 4`) bundles, taken directly from the `CodingKeys` enums in `Configuration/UTMQemuConfiguration.swift` and `Configuration/UTMAppleConfiguration.swift`:

```
Backend                  string  "QEMU" or "Apple"
ConfigurationVersion     int     4 in UTM 4.x
Information              dict    Name, Icon, IconCustom, Notes, UUID
System                   dict    Architecture, Target, MemorySize, CPUCount, …
Display                  array   one dict per virtual monitor
Drive                    array   one dict per disk / CD / BIOS / kernel
Network                  array   one dict per NIC
Serial                   array   one dict per serial device
Sound                    array   one dict per audio device
Sharing                  dict    QEMU only: DirectoryShareMode, DirectoryShareReadOnly, ClipboardSharing
Input                    dict    QEMU only: USB bus settings
QEMU                     dict    QEMU only: UEFIBoot, Hypervisor, AdditionalArguments, DebugLog, …
Virtualization           dict    Apple only: pointer device, audio, balloon, entropy, keyboard, rosetta
```

External-file references (drive sources outside the bundle, shared-directory paths) are stored as security-scoped bookmarks in UTM's `UserDefaults` under the `Registry` key — i.e. `~/Library/Containers/com.utmapp.UTM/Data/Library/Preferences/com.utmapp.UTM.plist`, **not** inline in `config.plist`. Hand-edits to `config.plist` cannot create new external references — use UTM's GUI or AppleScript `update registry` for that.

## QEMU configuration schema

Key fields (cross-referenced with `Configuration/UTMQemuConfigurationSystem.swift` etc. in the source):

### System

- `Architecture` (`x86_64`, `aarch64`, `arm`, `i386`, `riscv64`, `ppc64le`, `mips64`, `s390x`, …)
- `Target` (QEMU machine type, e.g. `q35`, `virt`, `pc-i440fx-8.0`)
- `MemorySize` (MiB)
- `CPUCount` (0 = match host)
- `ForceMulticore` (bool — keep multiple cores even when the guest expects single-core)
- `CPU` (e.g. `default`, `host`, `cortex-a72`)
- `CPUFlagsAdd`, `CPUFlagsRemove` (arrays of strings)
- `JITCacheSize` (MiB; 0 = default; iOS only)

### QEMU sub-dict

On disk these keys do **not** carry the `Has` prefix that the Swift property names use — the prefix is stripped in `CodingKeys`. Confirmed against `UTMQemuConfigurationQEMU.swift`:

- `UEFIBoot` (bool)
- `Hypervisor` (bool — HVF when host arch matches)
- `RTCLocalTime` (bool)
- `RNGDevice` (bool)
- `BalloonDevice` (bool)
- `TPMDevice` (bool — Windows 11 needs this)
- `TSO` (bool — Apple Silicon nested virtualization tweak)
- `MachinePropertyOverride` (string — extra `-machine ...` properties)
- `AdditionalArguments` (array of `qemu argument` dicts) — see [Custom QEMU arguments](#custom-qemu-arguments)
- `DebugLog` (bool)

### Drive

CodingKeys per `UTMQemuConfigurationDrive.swift`:

```
{
  "Identifier": "<uuid>",
  "ImageType": "Disk" | "CD" | "BIOS" | "LinuxKernel" | "LinuxInitrd" | "LinuxDTB" | "None",
  "Interface": "IDE" | "SCSI" | "SD" | "MTD" | "Floppy" | "PFlash" | "VirtIO" | "NVMe" | "USB" | "None",
  "InterfaceVersion": 1,
  "ImageName": "<file>.qcow2",      // present only for bundle-internal drives
  "ReadOnly": false
}
```

Notes:
- The plist key is `Interface`, not `InterfaceType`. Values are capitalized exactly as shown — `IDE` not `ide`, `VirtIO` not `virtio`.
- **`SATA` is not a valid value** in this enum. UTM exposes IDE/SCSI/VirtIO/NVMe/USB for typical disks.
- Drive size is **not** stored on disk — it is computed from the qcow2/raw file's actual size at load time.
- For drives that point at a host file outside the bundle (external ISO, etc.) the `ImageName` key is omitted; the file path is reconstructed from a bookmark in UTM's registry, not from `config.plist`.

### Network

CodingKeys per `UTMQemuConfigurationNetwork.swift`:

```
{
  "Mode": "Emulated" | "Shared" | "Host" | "Bridged",
  "Hardware": "virtio-net-pci" | "rtl8139" | "e1000" | …,
  "MacAddress": "52:54:00:…",
  "IsolateFromHost": false,
  "BridgeInterface": "en0",                 // Bridged mode
  "VlanGuestAddress": "10.0.2.0/24",        // Emulated mode (optional)
  "VlanHostAddress": "10.0.2.2",
  "VlanDhcpStartAddress": "10.0.2.15",
  "VlanDhcpEndAddress": "10.0.2.30",
  "VlanDhcpDomain": "internal",
  "VlanDnsServerAddress": "10.0.2.3",
  "VlanDnsSearchDomain": "internal",
  "HostNetUuid": "<uuid>",                  // Host mode (links VMs into one virtual network)
  "PortForward": [
    { "Protocol": "TCP",
      "GuestAddress": "",
      "GuestPort": 22,
      "HostAddress": "127.0.0.1",
      "HostPort": 2222 }
  ]
}
```

`Protocol` values are uppercase `"TCP"` and `"UDP"`. Mode values are capitalized `"Emulated"`/`"Shared"`/`"Host"`/`"Bridged"`.

### Display

CodingKeys per `UTMQemuConfigurationDisplay.swift`:

```
{
  "Hardware": "virtio-gpu-pci" | "virtio-gpu-gl-pci" | "qxl-vga" | "ramfb" | "vmware-svga" | …,
  "DynamicResolution": true,
  "NativeResolution": false,
  "UpscalingFilter": "Linear" | "Nearest",
  "DownscalingFilter": "Linear" | "Nearest",
  "VgaRamMib": 16                           // VGA RAM in MiB; on disk this key is VgaRamMib (not VgaRamSize)
}
```

### Sharing

CodingKeys per `UTMQemuConfigurationSharing.swift`:

```
{
  "DirectoryShareMode": "None" | "WebDAV" | "VirtFS",
  "DirectoryShareReadOnly": false,
  "ClipboardSharing": true
}
```

There is no `DirectoryShareBookmark` key — the bookmark to the host directory is stored in the registry, not the bundle.

## Apple configuration schema

The Apple backend records are simpler because Virtualization.framework hides QEMU-style detail. CodingKeys per `UTMAppleConfiguration*.swift`:

**`System.Boot`** (`UTMAppleConfigurationBoot.swift`):
- `OperatingSystem` — `Linux` | `macOS`
- `UEFIBoot` (bool — required for some Linux distros that boot via EFI rather than direct kernel)
- `LinuxKernelPath` (relative path inside `Data/`; the URL is reconstructed at load)
- `LinuxCommandLine` (kernel cmdline)
- `LinuxInitialRamdiskPath` (initrd, same path convention)
- `EfiVariableStoragePath` (NVRAM blob path)

The macOS recovery IPSW URL is not persisted (it's only used during install).

**Drives** (`UTMAppleConfigurationDrive.swift`, CodingKeys lines 39-44): `Identifier`, `ImageName` (bundle-internal drives), `Bookmark` (legacy field, kept for migration of older bundles), `ReadOnly`, `Nvme` (boolean — true exposes as NVMe, false as VirtIO Block). Drive size is **not** persisted as a plist key — it is read from the image file itself.

**Network** (`UTMAppleConfigurationNetwork.swift`, CodingKeys lines 46-48): `Mode` (`Shared` | `Bridged`), `MacAddress`, `BridgeInterface`. The Apple backend has no `Hardware` key — Virtualization.framework picks the device class itself.

**SharedDirectory** (`UTMAppleConfigurationSharedDirectory.swift`): `Bookmark` (security-scoped), `ReadOnly`. The host directory path itself is not stored as a string — it must be resolved through the bookmark.

**`Virtualization`** sub-dict (`UTMAppleConfigurationVirtualization.swift`, CodingKeys lines 67-74):
- `Audio`, `Balloon`, `Entropy` (booleans)
- `Keyboard` — enum string `"Disabled"` | `"Generic"` | `"Mac"` (capitalized exactly as shown)
- `Pointer` — enum string `"Disabled"` | `"Mouse"` | `"Trackpad"` (capitalized)
- `Trackpad` (legacy boolean kept for migration of pre-Pointer-enum configs)
- `Rosetta` (bool — macOS 13+ Apple Silicon only)
- `ClipboardSharing` (bool — host↔guest clipboard)

**Rosetta** mounts the host's x86_64 translator into a Linux guest under the virtiofs tag `rosetta` (not `share`). The guest mounts it with `mount -t virtiofs rosetta /mnt/rosetta` and registers it as a binfmt handler — see `UTMAppleConfigurationVirtualization.swift` for the VZ wiring.

## Modifying a bundle safely

Rules:

1. **UTM must not be running** while you edit `config.plist`. UTM rewrites the file on quit.
2. Always `cp -Rc` the bundle first, edit the copy, and re-import on success.
3. Round-trip through XML: `plutil -convert xml1 -o config.xml config.plist`, edit, then `plutil -convert binary1 -o config.plist config.xml` (UTM accepts either, but keeps it as XML by default).
4. Do not change `Backend` after a VM is created — the device arrays use different schemas and UTM will fail to load.
5. Do not change `Information.uuid` unless you also remove the bundle from UTM and re-import (UTM tracks VMs by UUID inside the registry).

A safer alternative for most fields: read the configuration via AppleScript, mutate, write back via `update configuration`. See [applescript.md](applescript.md#configuration-suite).

## Custom QEMU arguments

UTM exposes a free-form **QEMU Arguments** tab. Each entry is appended verbatim to the QEMU command line, after UTM's generated arguments. Useful examples:

- `-cpu host` — pass through full host CPU features (Intel host, Linux x86_64 guest).
- `-machine smm=off,vmport=off` — quiet certain Windows boot warnings.
- `-bios path/to/edk2.fd` — replace UTM's bundled UEFI firmware.
- `-monitor unix:/tmp/utm-mon,server,nowait` — expose a QMP/HMP socket so external tools can drive snapshots, hot-plug, or take screenshots.
- `-device usb-host,vendorid=0x046d,productid=0xc016` — pin a USB device by vendor/product (alternative to UTM's runtime connect/disconnect).

Order matters when arguments shadow each other. UTM does not validate the strings — a typo will cause QEMU to fail to launch with a console error in **Settings → QEMU → Debug Log**.

In the on-disk format these live under `QEMU.AdditionalArguments` as an array of **plain strings** (per `QEMUArgument.swift` — `init(from:)` decodes a `String` directly, not a keyed container). Example: `<array><string>-cpu host</string><string>-machine smm=off</string></array>`.

## Migrating a bundle between hosts

QEMU bundles are portable across Macs as-is — copy the bundle, double-click. Some pitfalls:

- **External drive bookmarks**. If a drive's `Bookmark` points outside the bundle (e.g. an ISO on the original Mac's Desktop), it will fail to resolve on the new host. Either move the file to the same path or re-attach the drive in UTM.
- **Bridged interface**. `Network[].BridgeInterface` is host-specific (`en0` may not exist on the other Mac). Edit before first run or switch to **Shared**.
- **Shared directory bookmarks**. Same problem as drives — re-add the share on the new host.
- **macOS guests** are tied to the Apple machine identity stored in `Data/HardwareModel`. They generally restore on another Apple Silicon Mac but Apple ID services may flag the move; consider this when migrating.

For a clean cross-host export, use **UTM → File → Export Selected** (`utmctl` has no export command — `osascript -e 'tell app "UTM" to export …'` does the job from CLI). Export resolves bookmarks into bundle-local copies where possible.
