# UTM end-user workflows

Recipes for the work users actually do in UTM: install a guest OS, set up file sharing, configure networking, manage snapshots/backups. Every workflow here assumes UTM 4.x on macOS. iOS is covered briefly in [troubleshooting.md](troubleshooting.md).

## Contents
- [Choosing a backend](#choosing-a-backend)
- [Where VMs live on disk](#where-vms-live-on-disk)
- [Importing a VM from the gallery](#importing-a-vm-from-the-gallery)
- [Installing Linux (Apple backend)](#installing-linux-apple-backend)
- [Installing Linux (QEMU backend)](#installing-linux-qemu-backend)
- [Installing Windows 11 ARM](#installing-windows-11-arm)
- [Installing Windows on Intel via QEMU](#installing-windows-on-intel-via-qemu)
- [Installing macOS as a guest](#installing-macos-as-a-guest)
- [File sharing](#file-sharing)
- [Networking](#networking)
- [Snapshots and backups](#snapshots-and-backups)
- [Adding a drive or ISO at runtime](#adding-a-drive-or-iso-at-runtime)

## Choosing a backend

| Use case | Backend |
| --- | --- |
| Run macOS guest on Apple Silicon | **Apple** (only option) |
| Run modern Linux fast on Apple Silicon | **Apple** preferred (virtio drivers, near-native speed) |
| Run Windows 11 ARM on Apple Silicon | **QEMU** (Apple Virtualization does not support Windows guests) |
| Run x86 Windows / Linux on Apple Silicon | **QEMU** with TCG emulation (slow) |
| Run x86 Windows / Linux on Intel Mac | **QEMU** with HVF acceleration (fast) |
| Need USB pass-through | **QEMU** |
| Need scripted keyboard / mouse input | **QEMU** |
| Need cross-arch (ARM, RISC-V, PPC, MIPS, …) | **QEMU** |
| Need fastest macOS-on-macOS | **Apple** |

The backend is permanent for a VM — you cannot convert in place. `duplicate` always preserves the backend.

## Where VMs live on disk

Default storage:

```
~/Library/Containers/com.utmapp.UTM/Data/Documents/<Name>.utm
```

`<Name>.utm` is a directory bundle (Finder hides this and shows a single icon — right-click → Show Package Contents). Inside:

```
<Name>.utm/
├── config.plist        # XML or binary plist; full config
├── Data/               # disk images and auxiliary storage
│   ├── <uuid>.qcow2    # virtual drives (or .img / .raw)
│   └── efi_vars.fd     # firmware NVRAM
├── view.plist          # last window size/position (per host)
└── screenshot.png      # last frame (used as VM tile)
```

The bundle is fully self-contained. Backup = `cp -R` (or `cp -Rc` on APFS for clone-reflinks). Move between Macs by copying the bundle and double-clicking it; UTM will register it.

To put VMs on an external drive, change the storage location in **UTM → Settings → General → Default location**, or move bundles by hand and double-click them on the new path.

## Importing a VM from the gallery

The official gallery is at <https://mac.getutm.app/gallery/>. Each entry links to a `.utm` or `.zip` containing one. To import:

1. Download and (if needed) unzip — it expands the qcow2 image, requires several GB free.
2. Double-click the `.utm` bundle, or drag onto UTM's main window, or `utmctl import` (no such command — use AppleScript: `tell application "UTM" to import new virtual machine from file …`).
3. First boot may take a minute as UTM expands sparse images.

Most gallery entries use the QEMU backend so they work on both Apple Silicon and Intel.

## Installing Linux (Apple backend)

Best path for fast, clean Linux on Apple Silicon. Limited to distros whose installer ships a Linux kernel + initrd UTM can launch directly (most modern x86_64 ISOs do not work; ARM64 ISOs do).

1. Download an ARM64 ISO (Ubuntu, Debian, Fedora — pick the `aarch64`/`arm64` server or live ISO).
2. **File → New** → **Virtualize** → **Linux**.
3. Pick the ISO. For Ubuntu/Debian server installers UTM autodetects the kernel and initrd from the ISO; if not, point it at them manually (mount the ISO and look in `casper/`, `install/`, or `boot/`).
4. Allocate RAM (4096+ MiB) and CPU (4 cores typical) and a disk (32+ GiB).
5. Optionally add a Shared Directory.
6. Boot, install as you would on bare metal. Keep the ISO attached only on first boot; remove afterward (Settings → Drives → CD/DVD).
7. After install, install GUI drivers if you want sharper graphics: `apt install spice-vdagent spice-webdavd` (or distro equivalent).

GPU acceleration via VirGL/Venus (`virtio-gpu-gl-pci`) works on recent macOS versions but is off by default; turn on **Display → GPU Supported** if your guest drivers are recent enough.

## Installing Linux (QEMU backend)

Use this when you need x86_64 Linux on Apple Silicon, or a non-mainstream architecture, or USB pass-through.

1. **File → New** → **Emulate** → **Linux** (or **Other** for exotic archs).
2. Pick architecture and machine. Defaults: `q35` for x86_64, `virt` for `aarch64`.
3. Tick **UEFI Boot** for modern distros.
4. Attach the ISO as a CD/DVD drive on first boot.
5. Allocate RAM (2048–4096 MiB), CPU (host cores), disk (32+ GiB).
6. **Sharing**: pick **VirtFS** (Linux-only, low overhead, `mount -t 9p -o trans=virtio share /mnt/share`) or **WebDAV / SPICE** (works for any guest with `spice-webdavd`).
7. Boot, install. Then install `qemu-guest-agent` so `utmctl exec`/`ip-address`/`file` work:
   ```
   sudo apt install -y qemu-guest-agent
   sudo systemctl enable --now qemu-guest-agent
   ```
8. Optionally install `spice-vdagent` for clipboard sharing and dynamic resolution.

x86_64 Linux on Apple Silicon runs at ~10–30 % of native through QEMU TCG. Acceptable for compiling/testing; not for desktop use.

## Installing Windows 11 ARM

Apple Silicon only. Install through the QEMU backend (the Apple backend cannot run Windows).

1. Generate a Windows 11 ARM ISO with [Crystalfetch](https://github.com/TuringSoftware/CrystalFetch) (UTM's sibling app, on the App Store) or download directly from Microsoft.
2. **File → New** → **Virtualize** → **Windows** (UTM detects the ARM architecture automatically).
3. Tick **Install drivers and SPICE tools** so virtio drivers and SPICE Guest Tools are mounted on first boot.
4. Allocate ≥ 4096 MiB RAM, ≥ 4 cores, ≥ 64 GiB disk.
5. Boot, run setup. When the partition step shows "no drives", click **Load driver** and pick `viostor` from the virtio CD.
6. After install, run the SPICE tools installer from the same CD for clipboard, dynamic resolution, and folder sharing.
7. Activate Windows or accept the eval period.

Common pitfalls:
- **Setup freezes on "Just a moment".** Hit **Shift+F10** to open `cmd`, run `OOBE\BYPASSNRO`, then continue. Lets you complete setup without a Microsoft account / network.
- **No mouse cursor in installer.** Some virtio versions miss the input driver — the SPICE tools CD includes it, install after first boot.
- **TPM/Secure Boot** are simulated by the QEMU machine; do not disable in config.

## Installing Windows on Intel via QEMU

On an Intel Mac the same x86_64 ISO works, with HVF acceleration enabled by default. Use the Windows preset, attach the ISO, install. Performance is near-native.

On Apple Silicon you can still run x86_64 Windows via QEMU TCG — installation alone may take ≥ 1 hour and runtime is slow. Prefer Windows 11 ARM unless you specifically need x86 Windows software (in which case Microsoft's x86 emulation inside Windows 11 ARM is usually faster than QEMU TCG).

## Installing macOS as a guest

Apple Silicon host only, macOS 12+ host, Apple backend only. Cannot run on Intel Macs.

1. **File → New** → **Virtualize** → **macOS 12+**.
2. UTM offers to fetch the latest IPSW restore image automatically, or accept a path you specify.
3. Allocate RAM (4096+ MiB) and disk (≥ 64 GiB; 128 GiB+ if installing Xcode).
4. UTM creates an auxiliary storage and machine-identity blob inside the bundle automatically.
5. Boot — first launch installs macOS into the disk image, takes 10–20 minutes.
6. Complete the macOS Setup Assistant inside the guest.

Limitations of macOS guests:
- No GPU acceleration (no Metal). 3D apps run on CPU.
- iCloud account sign-in works but Apple ID activation may fail; some users keep guests local-only.
- USB pass-through is not available (Apple backend limitation).
- Cloning a macOS guest produces two VMs with the same machine identity — Apple's services may flag the second one. Use `duplicate` (which regenerates the auxiliary blob) rather than copying the bundle.

## File sharing

| Backend | Recommended share method | Mount inside guest |
| --- | --- | --- |
| Apple, Linux/macOS guest | Apple-native shared directory | Auto-mounts as `/Volumes/My Shared Files` (macOS guest) or via `mount -t virtiofs share /mnt/share` (modern Linux) |
| QEMU, Linux guest | VirtFS (9P) | `mount -t 9p -o trans=virtio,version=9p2000.L share /mnt/share` |
| QEMU, Windows or other | WebDAV via SPICE Guest Tools | Tools install a `Spice client folder` mapping; or `\\spice-host\Shared` |
| Either, big files | SMB/SSH from host network | Standard SMB/SSH client in guest |

WebDAV requires `spice-webdavd` in the guest (Linux) or the SPICE Tools installer (Windows). VirtFS requires the host to expose a directory in **Settings → Sharing**.

Apple Virtualization shared directories are read/write by default. To share read-only, tick "Read Only" when adding the share.

## Networking

| Mode | Effect |
| --- | --- |
| **Shared (NAT)** | Default. Guest gets a private IP from UTM's DHCP, can reach the internet, host can reach guest. Works without privacy prompts. |
| **Bridged** | Guest is a peer on the host's LAN. Pick a host interface (en0/Wi-Fi). On macOS Sequoia+, requires granting UTM "Local Network" permission. |
| **Host-Only** | Guest is isolated to a virtual subnet shared with other UTM guests on this host. No external connectivity. |
| **Emulated VLAN** | QEMU only; the SLIRP user network with full DHCP/DNS configurable. Useful for offline labs. |

### Port forwarding (Shared mode, QEMU only)

Per-VM, **Settings → Network → New Port Forward**. UTM stores them in `additionalArguments` as QEMU `-netdev hostfwd=…` entries. Host port → Guest IP : Guest port.

```
host TCP 2222 → 192.168.64.x:22       # SSH from host: ssh -p 2222 user@localhost
host TCP 8080 → 192.168.64.x:80
```

Apple-backend VMs do not support port forwarding — use Bridged mode and connect to the guest's LAN IP, or set up a reverse SSH tunnel from inside the guest.

### Finding the guest's IP

- QEMU + guest agent: `utmctl ip-address "<vm>"`
- Apple backend: hover over the network icon in UTM's status bar; or look up the MAC in the host's ARP table (`arp -a | grep <mac>`).
- mDNS: most distros announce themselves at `<hostname>.local`.

## Snapshots and backups

UTM exposes only a primitive snapshot model:

- **Suspend with save state** (`utmctl suspend --save-state`) writes the running RAM/CPU snapshot into the bundle. Resume with `utmctl start`.
- **Disposable mode** (`utmctl start --disposable`) runs from a scratch overlay; changes are discarded.
- **Manual snapshots** are not in the CLI/AppleScript dictionary. The QEMU monitor command `savevm`/`loadvm` works if you build your own QEMU monitor connection (advanced).

For real backups, stop the VM and copy the bundle. APFS clones make this near-instant:

```bash
cd ~/Library/Containers/com.utmapp.UTM/Data/Documents
cp -Rc "Ubuntu.utm" /Volumes/Backups/Ubuntu.utm
```

Use `fsfreeze` (Linux) or `vssadmin Create Shadow` (Windows) before copying a running guest's bundle if you do not want to stop the VM. See the snapshot recipe in [applescript.md](applescript.md#take-a-snapshot-via-guest-side-fsfreeze-no-utm-native-snapshot-cli).

## Adding a drive or ISO at runtime

A drive must be added while the VM is **stopped**:

1. Stop the VM.
2. **Settings → Drives → New** → choose Disk Image, ISO, or NVMe/IDE/SCSI/VirtIO; pick a host file.
3. Save and restart.

Removable drives (CD/DVD) can be **swapped at runtime** via the toolbar disk icon — UTM updates the registry bookmark; the underlying QEMU sees a media-change event. Use **AppleScript `update registry`** to do the same from a script.

For per-VM customization beyond the GUI, edit `config.plist` while UTM is closed. See [configuration.md](configuration.md).
