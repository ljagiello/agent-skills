# utmctl reference

`utmctl` is the CLI bundled with UTM.app. It is implemented in Swift on top of the AppleScript bridge, so every operation here also has an AppleScript equivalent — but the CLI is shorter for shell scripting.

## Contents
- [Locating and invoking utmctl](#locating-and-invoking-utmctl)
- [Global behavior](#global-behavior)
- [Identifiers](#identifiers)
- [version](#version)
- [list](#list)
- [status](#status)
- [start](#start)
- [suspend](#suspend)
- [stop](#stop)
- [clone](#clone)
- [delete](#delete)
- [attach](#attach)
- [ip-address](#ip-address)
- [exec](#exec)
- [file pull / file push](#file)
- [usb list / connect / disconnect](#usb)
- [Common patterns](#common-patterns)

## Locating and invoking utmctl

`utmctl` lives inside the app bundle:

```
/Applications/UTM.app/Contents/MacOS/utmctl
```

Either call it by full path or symlink it once:

```bash
sudo ln -sf /Applications/UTM.app/Contents/MacOS/utmctl /usr/local/bin/utmctl
```

UTM does not need to be running — `utmctl` will launch it on demand. When invoked from the App Store build with a tightened sandbox, the first call may produce a permission prompt asking the user to allow scripting of UTM; this must be approved interactively.

`utmctl` is unusable from a plain SSH session because the AppleScript send fails outside an Aqua login. From a SSH session you can `osascript -e 'tell application "UTM" to launch'` after running `caffeinate` or after using `screen sharing` to log in.

## Global behavior

```
utmctl [--debug] [--hide] <subcommand> ...
```

- `-d`, `--debug` — print debug output to stderr.
- `--hide` — keep UTM windows hidden after the command runs.
- Exit code 0 on success, non-zero on error. Most failures print a localized `NSError` description to stderr.
- All commands operate over the AppleScript bridge; they are synchronous from the CLI's perspective but the underlying VM action may continue (e.g. `start` returns once UTM has dispatched the start, not once the guest has booted).

## Identifiers

Wherever an `<identifier>` is required, you can pass either:

- The VM's exact `name` (case-sensitive). Quote names containing spaces.
- The VM's `id` UUID as printed by `utmctl list`.

Names are convenient for humans; UUIDs are stable across rename. Prefer UUIDs in long-lived scripts.

---

## version

```
utmctl version
```

Prints UTM's version string (e.g. `4.6.5`). Useful as a probe in CI to confirm UTM is installed and scriptable.

## list

```
utmctl list
```

Prints a header followed by one row per registered VM:

```
UUID                                 Status   Name
3f1b2c0a-…-… stopped  Ubuntu 24.04
9e8a7d54-…-… started  Windows 11 ARM
```

`Status` is one of: `stopped`, `starting`, `started`, `pausing`, `paused`, `resuming`, `stopping`. Parse this with awk on whitespace, or grep by UUID.

## status

```
utmctl status <identifier>
```

Prints one of the status enumerators above. Exits non-zero if the VM cannot be found.

## start

```
utmctl start [-a | --attach] [--disposable] [--recovery] <identifier>
```

- `--disposable` — equivalent to QEMU's `-snapshot`: writes go to a scratch overlay and are discarded when the VM stops. Excellent for CI runs.
- `--recovery` — boot the VM into recovery mode. Required to reinstall macOS guests; ignored on most Linux configurations.
- `-a`, `--attach` — accepted but the post-start serial attach is unimplemented. The VM still starts normally; utmctl prints `WARNING: attach command is not implemented yet!` to stdout via plain `print()`. Treat the flag as a no-op.

`start` resumes a suspended VM as well as cold-starting a stopped one.

## suspend

```
utmctl suspend [--save-state] <identifier>
```

- Without `--save-state`, the VM is paused in memory only. Quitting UTM (or rebooting the host) loses the state.
- With `--save-state`, UTM writes a snapshot to disk inside the `.utm` bundle, so the VM can be cold-resumed later.

## stop

```
utmctl stop [--force | --kill | --request] <identifier>
```

Mutually exclusive flags; default is `--force`:

- `--force` (default) — sends a stop request to the QEMU/VZ backend. Equivalent to "Stop" in the menu. Fast, may corrupt unflushed guest writes the same way pulling power would.
- `--request` — issues an ACPI/QMP power-down request. The guest OS sees this as the user pressing the power button and shuts down cleanly. Some guests (or guests with no power-button handler) may ignore it indefinitely.
- `--kill` — terminates the QEMU process. Last resort for hung VMs.

For automated test runs prefer `--request` followed by polling `utmctl status` until `stopped`, with a fallback to `--force`.

## clone

```
utmctl clone [--name <new-name>] <identifier>
```

Duplicates the VM bundle (deep copy of disks). If `--name` is omitted, UTM derives a name like `<original> Clone`. The clone's UUID is freshly generated.

## delete

```
utmctl delete <identifier>
```

**No confirmation.** Deletes the bundle from disk and removes it from the registry. There is no undo. Always print and confirm `utmctl list` output before scripting `delete`.

## attach

```
utmctl attach [--index <N>] <identifier>
```

Intended to redirect a serial port to the calling terminal. The terminal-emulation half is unimplemented, but the command **does print the serial endpoint** before returning, so it is usable as a discovery tool. Output looks like:

```
WARNING: attach command is not implemented yet!
PTTY: /dev/ttys009
```

or, for a TCP serial port:

```
WARNING: attach command is not implemented yet!
TCP: 127.0.0.1:4001
```

`--index N` selects the serial-port index (defaults to the first one with an available interface). Pipe the output through `grep -E '^(PTTY|TCP):'` to extract the address, then connect with `screen <ptty>` or `nc <host> <port>`. For an AppleScript-based alternative that does not print a warning, see [applescript.md](applescript.md#serial-ports).

## ip-address

```
utmctl ip-address <identifier>
```

Prints one IP per line. IPv4 addresses appear before IPv6. Loopback addresses are excluded. Requires the **QEMU guest agent** to be running in the guest. Will fail on the Apple backend (no guest agent).

Useful pattern (poll until ready):

```bash
until ip=$(utmctl ip-address "Ubuntu" 2>/dev/null | head -n1) && [ -n "$ip" ]; do
    sleep 2
done
echo "Guest IP: $ip"
```

## exec

```
utmctl exec [--input] [--env NAME=VALUE]... <identifier> -- <command> [args...]
```

Executes `command` inside the guest via the QEMU guest agent.

- `--input` — read host stdin and forward to the guest process's stdin.
- `--env NAME=VALUE` — repeat for additional env vars. UTM passes these as a flat list.
- Use `--` before the guest command so utmctl's argument parser stops consuming flags — without it, a guest flag like `-c` is parsed as a utmctl flag and rejected.

Output behavior:

- Guest stdout streams to host stdout, guest stderr to host stderr.
- Exit status of the guest process becomes `utmctl`'s exit status.
- Output is captured on the UTM side until the guest process exits, then flushed; you do not get true real-time streaming. For long-running processes prefer SSH.

Examples:

```bash
# Capture command output
out=$(utmctl exec "Ubuntu" -- /usr/bin/uname -srm)

# Forward stdin
echo 'print("hi")' | utmctl exec --input "Ubuntu" -- /usr/bin/python3 -

# Set env
utmctl exec --env LC_ALL=C --env DEBIAN_FRONTEND=noninteractive \
    "Ubuntu" -- /usr/bin/apt-get -y update
```

Failure modes:

- Returns "operation not supported" on Apple-backend VMs (no QEMU guest agent).
- Hangs if the guest agent service isn't running. Inside Linux: `systemctl status qemu-guest-agent`. Inside Windows: ensure the `QEMU Guest Agent VSS Provider` service is running.
- The path passed to `at`/the executable must exist or be reachable through the guest's `PATH`. The agent does not run a shell unless you call one explicitly (`/bin/sh -c '...'`).

<a name="file"></a>
## file pull / file push

```
utmctl file pull <identifier> <guest-path>
utmctl file push <identifier> <guest-path>
```

`pull` writes the guest file's contents to host stdout. `push` reads host stdin and writes it to `<guest-path>` on the guest. Both transfer in 4096-byte base64-encoded chunks via the guest agent, so they handle binary data correctly but are slow for files larger than a few MB — for big transfers, use SSH/SCP/rsync over the guest's IP instead.

```bash
# Save guest log
utmctl file pull "Ubuntu" /var/log/syslog > syslog.txt

# Upload a file
utmctl file push "Ubuntu" /root/setup.sh < ./setup.sh
utmctl exec "Ubuntu" -- /bin/sh -c 'chmod +x /root/setup.sh && /root/setup.sh'
```

The guest path must be writable by the user the guest agent runs as (typically root on Linux, SYSTEM on Windows).

<a name="usb"></a>
## usb list / connect / disconnect

```
utmctl usb list
utmctl usb connect <vm-identifier> <device-identifier>
utmctl usb disconnect <device-identifier>
```

`usb list` prints currently-attached host USB devices that UTM can see, with columns `Name`, `VID :PID` (note the space before the colon — that is the literal column header), and `Location`. The device identifier passed to `connect`/`disconnect` is either:

- `VID:PID` in hex (e.g. `046D:C016`), or
- the integer Location id from `usb list`.

USB pass-through is **QEMU-only**. The Apple Virtualization backend cannot attach host USB devices. Some hosts also require granting UTM the "Input Monitoring" or "USB" entitlement before the device shows up.

```bash
# Move a YubiKey into a Windows VM, work, then return it to the host
utmctl usb connect "Windows 11" 1050:0407
# … VM uses the device …
utmctl usb disconnect 1050:0407
```

---

## Common patterns

### Wait for a state change

```bash
wait_for() {
    local vm=$1 want=$2 timeout=${3:-120}
    for ((i=0; i<timeout; i++)); do
        [ "$(utmctl status "$vm")" = "$want" ] && return 0
        sleep 1
    done
    return 1
}

utmctl start "Ubuntu"
wait_for "Ubuntu" started
```

### Graceful stop with fallback

```bash
graceful_stop() {
    local vm=$1
    utmctl stop --request "$vm"
    if ! wait_for "$vm" stopped 60; then
        echo "Guest ignored power-down, forcing." >&2
        utmctl stop --force "$vm"
    fi
}
```

### CI: run a command in a disposable clone

```bash
utmctl clone "Ubuntu Base" --name "ci-$BUILD_ID"
utmctl start --disposable "ci-$BUILD_ID"
trap 'utmctl stop --force "ci-$BUILD_ID"; utmctl delete "ci-$BUILD_ID"' EXIT
# wait for guest agent
until utmctl ip-address "ci-$BUILD_ID" >/dev/null 2>&1; do sleep 2; done
utmctl exec "ci-$BUILD_ID" -- /bin/bash -lc './run-tests.sh'
```

### Discover all VMs in a script

```bash
utmctl list | tail -n +2 | awk '{ uuid=$1; status=$2; $1=$2=""; sub(/^  /,""); print uuid"\t"status"\t"$0 }'
```

The first column is always a UUID, second is the status, the rest is the name (which can contain spaces).

### Detect backend before sending input

```bash
backend=$(osascript -e 'tell application "UTM" to get backend of virtual machine named "MyVM" as text')
[ "$backend" = "qemu" ] || { echo "Input automation requires QEMU backend" >&2; exit 1; }
```

For input itself, see [applescript.md](applescript.md#input-automation) — `utmctl` has no `input` subcommand.

## Things utmctl does NOT do

These require AppleScript:

- Create a new VM (`make new virtual machine`).
- Send keyboard input or mouse clicks (`input keystroke`, `input mouse click`, `input scan code`).
- Read or change configuration (RAM, CPU, disks, network mode, port forwards, displays).
- Open or read/write guest files at arbitrary offsets — `utmctl file` always streams from offset 0 and closes after.
- Mount/unmount removable media or update directory-share bookmarks (`update registry`).

For all of those, see [applescript.md](applescript.md).
