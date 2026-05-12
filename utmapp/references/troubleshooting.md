# UTM troubleshooting and edge cases

Read this when something does not work. Topics are grouped; each entry leads with the symptom.

## Contents
- [Scripting / utmctl errors](#scripting--utmctl-errors)
- [Guest agent (exec/file/ip-address) failures](#guest-agent-execfileip-address-failures)
- [Networking issues](#networking-issues)
- [Performance problems](#performance-problems)
- [Display, GPU, and resolution](#display-gpu-and-resolution)
- [USB pass-through](#usb-pass-through)
- [Snapshots and save state](#snapshots-and-save-state)
- [iOS / iPadOS specifics](#ios--ipados-specifics)
- [Build-from-source pitfalls](#build-from-source-pitfalls)

## Scripting / utmctl errors

**Symptom: `Application can't be found.` or `Not authorized to send Apple events to UTM.`**
The caller does not have automation permission. Open **System Settings → Privacy & Security → Automation**, find your terminal / script runner, and tick UTM. There is no programmatic way to grant this.

**Symptom: Commands hang or return generic errors when run via SSH.**
AppleScript needs a logged-in graphical session. Solutions in order of preference:

1. Run in Terminal/Tmux inside Screen Sharing.
2. Use a launchd LaunchAgent (loaded at login).
3. `caffeinate -d -u` keeps the session active without sleep.
4. Last resort: configure auto-login on the host.

**Symptom: `utmctl: command not found`.**
The CLI is not on `PATH`. Either call it by full path:
```
/Applications/UTM.app/Contents/MacOS/utmctl …
```
or symlink it: `sudo ln -sf /Applications/UTM.app/Contents/MacOS/utmctl /usr/local/bin/utmctl`. The App Store build is at the same path.

**Symptom: `utmctl attach` says "not yet implemented".**
Correct — the flag exists but the implementation is incomplete. Read the serial port's `address`/`port` via AppleScript and connect with `screen` or `nc`. See [applescript.md](applescript.md#serial-ports).

**Symptom: `utmctl exec` returns immediately with no output.**
Likely capturing was off in the underlying call (utmctl sets it on by default, but if you build a custom AppleScript skip this), or the guest command exited before output flushed. Add explicit redirection: `utmctl exec "$vm" -- /bin/sh -c 'cmd 2>&1'`.

**Symptom: `utmctl start` prints `OSStatus error -2700 / Operation not available` against an Apple-backend VM, but `utmctl status` says `started`.**
Cosmetic. `OSStatus -2700` is the generic AppleScript "event failed" envelope; the real meaning is in the trailing message and the resulting VM state. `UTMScriptingVirtualMachineImpl.start` first attaches a window controller (`data.run(vm:startImmediately:false)`) and then re-reads `vm.state` — on Apple-backend VMs the state has often already left `.stopped`, so the bridge throws `operationNotAvailable` even though the start succeeded.

Do **not** retry the start, delete the VM, or treat the non-zero exit code as authoritative. Verify with `utmctl status "<vm>"`; if it returns `starting` or `started`, continue. Same pattern when scripting `start` via AppleScript directly.

**Symptom: `utmctl exec` / `utmctl file` / `utmctl ip-address` prints `OSStatus error -2700 / Operation not supported by the backend`.**
Real failure — and it will keep failing. This is the *other* `-2700` variant: the Apple Virtualization backend has no QEMU guest agent, so these commands have nothing to talk to. Pivot to SSH (`exec`, `file`) or ARP/mDNS on `bridge100` (`ip-address`). See [SKILL.md → Finding a guest's IP](../SKILL.md#finding-a-guests-ip). The two `-2700` cases are distinguished only by the trailing message line — always read it.

## Guest agent (exec/file/ip-address) failures

**Symptom: `query ip` returns empty list, or exec hangs.**
The QEMU guest agent is not running. Check inside the guest:

- Linux: `systemctl status qemu-guest-agent`. Install with `apt install qemu-guest-agent` or distro equivalent. The agent talks over a virtio-serial port that UTM auto-creates; you do not need to add anything in the host config.
- Windows: install **virtio-win Guest Tools**. Verify the `QEMU Guest Agent` service is running.

**Symptom: agent runs but `exec` reports "operation not supported".**
The VM uses the Apple Virtualization backend, which has no QEMU guest agent. Use SSH instead, or convert the workflow to AppleScript-only operations.

**Symptom: `file pull` corrupts binary files.**
You forgot `--binary` … there is no such flag. `utmctl file pull` already streams binary data through base64 internally, so this should not happen — but make sure no shell interprets the bytes (redirect to a file with `>` rather than piping through `read`/`xargs`).

## Networking issues

**Symptom: Bridged guest never gets an IP on macOS Sequoia.**
macOS 15 added a "Local Network" privacy permission that UTM needs to bridge. **System Settings → Privacy & Security → Local Network → UTM ✓**. After the first refusal you may need to remove and re-add UTM in the list.

**Symptom: Port forwarding works, but only `127.0.0.1`-bound services on the guest are reachable.**
The QEMU SLIRP user network's port forward terminates inside the guest's NIC. If the guest service binds to `127.0.0.1` it will not see the forwarded packets — bind to `0.0.0.0` or to the guest's NIC IP.

**Symptom: Two QEMU VMs in "Host" mode cannot ping each other.**
Both VMs need to be on the **same** host-only network. UTM creates one shared host-only subnet, so this should work — verify both VMs have `Network.Mode = "Host"` in `config.plist` and not `Emulated`.

**Symptom: Apple-backend Linux guest has no IPv6 / no internet.**
Apple's shared mode (`vmnet-shared`) does NAT44 only. For IPv6 you need Bridged mode plus Local Network permission.

**Symptom: Slow DNS / failed lookups on QEMU SLIRP networks.**
SLIRP forwards DNS to the host via the synthetic `10.0.2.3` resolver. If the host's DNS is going through a VPN that disallows split tunneling, lookups can fail. Switch to Bridged or set explicit DNS in the guest (`8.8.8.8`, `1.1.1.1`).

## Performance problems

**Symptom: x86_64 Linux/Windows on Apple Silicon is unbearably slow.**
That's QEMU TCG translating x86 instructions on the fly. Expectations: 10–30 % of native. Mitigations:

- Use ARM64 Linux / Windows 11 ARM where possible.
- Inside Windows 11 ARM, run x86 apps under Microsoft's built-in x86 emulator (faster than QEMU TCG nested in QEMU).
- For Linux x86_64 specifically, mount **Rosetta** in an ARM Linux guest under the Apple backend (Settings → Sharing → "Run x86 binaries through Rosetta"). Single ARM kernel, ARM and x86_64 user-space binaries both run.
- Reduce the working set: fewer cores often runs faster than more, because TCG does not parallelize per-vCPU efficiently.

**Symptom: Apple-backend macOS guest sluggish in graphical apps.**
No GPU acceleration is exposed by Virtualization.framework — even simple animations are CPU-rendered. There is no fix; for graphics-heavy macOS work, run on the host.

**Symptom: QEMU guest pegs CPU even when idle.**
Common with old Linux kernels lacking PV interrupt drivers, or with the `cirrus` display in graphical mode. Switch the display to `virtio-gpu-pci`, install `qemu-guest-agent` (it implements idle hints), and ensure `tickless` is on in the guest kernel.

**Symptom: HVF is supposedly enabled but performance feels emulated.**
HVF only kicks in when host arch == guest arch. Check `Architecture` in `config.plist`: it must match `arm64`/`aarch64` on Apple Silicon, or `x86_64` on Intel. Also verify `QEMU.HasHypervisor = true`.

## Display, GPU, and resolution

**Symptom: Resolution does not change when I resize the window.**
Dynamic resolution requires the SPICE guest agent (`spice-vdagent`) on Linux, or the SPICE Tools installer on Windows. Without it, the resolution is whatever the OS chose at boot.

**Symptom: Linux guest has black screen after install.**
Most often the bootloader is still pointing at a virtual console only. Switch the display device to `virtio-gpu-pci` (or `virtio-gpu-gl-pci` for VirGL) in **Settings → Display**, or boot with `console=tty0 console=ttyS0` so output also goes to the serial port and you can debug there.

**Symptom: Tiny / blurry text on Retina display.**
Tick **Settings → Display → Native Resolution** for sharp 2× rendering, then bump the guest's DPI / scale factor. Without "Native Resolution", UTM hands the guest the logical (low-DPI) size.

## USB pass-through

**Symptom: `utmctl usb connect` says "no such device".**
- Check `utmctl usb list` first; the host may not yet see the device if it's still enumerating.
- Some devices require **Input Monitoring** permission for UTM (mice/keyboards in particular).
- USB pass-through is **QEMU only**; the Apple backend never lists devices.

**Symptom: Device connects, then disconnects after a few seconds.**
Power-management timeouts in QEMU EHCI/XHCI. Add a `-device qemu-xhci` (USB 3) explicitly via custom QEMU arguments and connect again — often more reliable than the default USB 2 hub UTM uses.

**Symptom: Apple Silicon kernel panic when connecting USB to a Linux guest.**
Known interaction between certain USB 3 hubs and the macOS USB stack. Plug the device into a different port (preferably the Mac's built-in port, not a hub), or use a USB-IP server on the host instead.

## Snapshots and save state

**Symptom: Suspending a VM with `--save-state` fails with "device does not support live save".**
QEMU live state save requires every device in the VM to be migration-aware. The usual culprit is a USB host-pass-through device: disconnect first (`utmctl usb disconnect …`), then suspend.

**Symptom: After resume, guest network is dead.**
DHCP leases time out across long suspends. Inside the guest, `dhclient -r && dhclient` (Linux) or `ipconfig /release && ipconfig /renew` (Windows). Apple-backend macOS guests handle this transparently.

**Symptom: I want a real "snapshot tree" like VirtualBox.**
UTM does not expose `savevm`/`loadvm` in the GUI. Workarounds:

1. Stop the VM and `cp -Rc bundle.utm bundle-state-1.utm` for an APFS reflink copy. Restoration = swap the bundle back.
2. Use a custom QEMU monitor argument (`-monitor unix:…`) and drive `savevm` / `loadvm` from a script.
3. Maintain qcow2 backing-chain images outside UTM (advanced; UTM does not always preserve the chain).

## iOS / iPadOS specifics

UTM on iOS comes in two flavors:

- **UTM (full)** — uses JIT for QEMU TCG. Requires either a jailbroken iOS (Palera1n, Dopamine), an enterprise certificate (rare), or a runtime JIT enabler (AltStore / SideStore + JitStreamer / StikJIT) which works only on specific iOS versions.
- **UTM SE** ("Slow Edition") — uses a threaded interpreter instead of JIT, no special privilege required. Available on the App Store. ~3× slower than JIT for compute-heavy workloads but fine for terminal-only use.

Common questions:

- "Can I sideload UTM with regular AltStore?" — Yes, but JIT will not work without a JIT enabler app on the same device. Without JIT, performance is uselessly slow; install UTM SE instead.
- "How do I refresh the 7-day cert?" — Open AltStore weekly while connected to the same Wi-Fi as your AltServer. SideStore can refresh fully on-device (no Mac needed) using a wireguard-based proxy.
- "Why does my VM die when iOS backgrounds?" — iOS reclaims memory aggressively. Pin UTM to the foreground or accept that long suspends will kill the VM.

There is no `utmctl`, no AppleScript, and no shell on iOS — automation is not supported. The skill's CLI/AppleScript content does not apply on iOS.

## Build-from-source pitfalls

These come up when users try to build UTM themselves to get debug logging or experiment with the source.

- **Xcode signing**. The `Build.xcconfig` template has placeholders; copy `CodeSigning.xcconfig.sample` to `CodeSigning.xcconfig` and fill in your team ID before opening the project.
- **Submodule depth**. `git clone --recursive` is required; QEMU and SPICE patch trees are submodules.
- **Build time on M1 Air**. Around 30 minutes for the first full build (mainly QEMU and SPICE-related libraries). Subsequent builds are minutes.
- **Tethered launch (jailbroken iOS)**. The app must be re-launched via a paired Mac after every reboot; see `Documentation/TetheredLaunch.md` in the source tree.

For deeper development questions, point users at `Documentation/MacDevelopment.md` and `Documentation/iOSDevelopment.md` in the upstream repo (<https://github.com/utmapp/UTM>) rather than answering inline.
