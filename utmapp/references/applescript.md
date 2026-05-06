# UTM AppleScript / JXA reference

UTM ships a complete AppleScript dictionary (`UTM.sdef` inside the app bundle). Open it in Script Editor (File → Open Dictionary → UTM) for the live, browsable version. This file documents the surface relevant to automation.

Everything below works with both classic AppleScript and JXA (`osascript -l JavaScript`). JXA is recommended when the surrounding code is also JavaScript or when manipulating records (configuration dictionaries) is easier with object literals.

## Contents
- [Suites overview](#suites-overview)
- [Application object](#application-object)
- [Lifecycle commands](#lifecycle-commands)
- [Creating, importing, exporting, cloning, deleting](#creating-importing-exporting-cloning-deleting)
- [Configuration suite](#configuration-suite)
- [Guest agent suite](#guest-agent-suite)
- [Input automation](#input-automation)
- [USB suite](#usb-suite)
- [Registry suite](#registry-suite)
- [Serial ports](#serial-ports)
- [JXA recipe collection](#jxa-recipe-collection)

## Suites overview

| Suite | Purpose | Backend support |
| --- | --- | --- |
| UTM Suite | Core lifecycle: start/suspend/stop/delete/duplicate/import/export | both |
| UTM Guest Suite | `execute`, `query ip`, `open file`, `read`, `write`, `pull`, `push`, `close`, `get result` | QEMU only (requires guest agent) |
| UTM Configuration Suite | Read `configuration of …`, `update configuration` | both, but record schemas differ |
| UTM USB Devices Suite | List + `connect` / `disconnect` USB devices | QEMU only |
| UTM Registry Suite | `update registry` (rebind shared dirs / drive bookmarks) | both |
| UTM Input Automation Suite | `input scan code`, `input keystroke`, `input mouse click` | QEMU only |

The suite's access group is `com.utmapp.UTM.vm-access` — the calling process must be granted UTM scripting access (the user gets a TCC prompt the first time).

## Application object

```applescript
tell application "UTM"
    UTM version           -- text, read-only
    auto terminate        -- boolean, read/write
    virtual machines      -- list of "virtual machine" specifiers
    usb devices           -- list of "usb device" specifiers
end tell
```

Reference a VM by name or by index:

```applescript
tell application "UTM" to set vm to virtual machine named "Ubuntu"
tell application "UTM" to set vm to first virtual machine whose id is "3f1b2c0a-…-…"
```

## Lifecycle commands

```applescript
start <vm> [saving <bool>] [recovery <bool>]
suspend <vm> [saving <bool>]
stop <vm> [by force | kill | request]
delete <vm>          -- no confirmation
duplicate <vm> [with properties {configuration:{name:"new"}}]
import new virtual machine from file <hfs-path>
export <vm> to file <hfs-path>
```

`saving` defaults to `true` for `start` and `false` for `suspend`. `recovery` defaults to `false`. `by` defaults to `force`. The `status` property reflects the result and can be polled.

```applescript
tell application "UTM"
    set vm to virtual machine named "Ubuntu"
    start vm                                     -- cold start or resume
    repeat while (status of vm) is not started
        delay 1
    end repeat
    stop vm by request
end tell
```

## Creating, importing, exporting, cloning, deleting

```applescript
tell application "UTM"
    set newVM to make new virtual machine with properties {¬
        backend: qemu, ¬
        configuration: {name:"Test", architecture:"aarch64", memory:2048}}
end tell
```

`backend` is one of `qemu`, `apple`, or `unavailable`. For QEMU you must specify at least `name` and `architecture` in the configuration record; for Apple you specify `name`. The schemas of the configuration record are below.

`import` accepts an HFS file path to a `.utm` bundle:

```applescript
tell application "UTM" to import new virtual machine from file "/Users/me/Downloads/Ubuntu.utm"
```

`export` writes a `.utm` bundle copy:

```applescript
tell application "UTM" to export virtual machine named "Ubuntu" to file "/Users/me/Backups/Ubuntu.utm"
```

`duplicate` returns a new VM specifier and (optionally) renames it:

```applescript
tell application "UTM" to duplicate virtual machine named "Ubuntu" ¬
    with properties {configuration:{name:"Ubuntu Clone"}}
```

## Configuration suite

```applescript
configuration of <vm>           -- record (read-only); shape depends on backend
update configuration <vm> with <record>   -- VM must be stopped
```

`update configuration` cannot change the backend.

### qemu configuration record

Top-level fields (per `UTM.sdef`):

- `name`, `icon`, `notes` — text
- `architecture` — text (e.g. `x86_64`, `aarch64`, `ppc64`, `riscv64`)
- `machine` — QEMU machine type (e.g. `q35`, `virt`)
- `memory` — integer MiB
- `cpu cores` — integer (0 = default)
- `hypervisor` — boolean (HVF when host arch matches)
- `uefi` — boolean
- `directory share mode` — `none` | `WebDAV` | `VirtFS`
- `drives` — list of `qemu drive configuration` records (`id`, `interface`, `host size`, `guest size`, `raw`, `source`, `removable`)
- `network interfaces` — list of `qemu network configuration` records (`hardware`, `mode` ∈ `emulated`/`shared`/`host`/`bridged`, `address` (MAC), `host interface` for bridged, `port forwards`)
- `serial ports` — list of `qemu serial configuration` records (`hardware`, `interface`, `port`)
- `displays` — list of `qemu display configuration` records (`hardware`, `dynamic resolution`, `native resolution`, `upscaling filter`, `downscaling filter`)
- `qemu additional arguments` — list of `qemu argument` records (raw QEMU CLI args; each has `argument string` and optional `file urls`)

Port-forward sub-record fields: `protocol` (`TCP`/`UDP`), `host address`, `host port`, `guest address`, `guest port`.

**Important AppleScript naming quirk**: the additional-arguments property is `qemu additional arguments`, not `additional arguments`. Reading or writing the wrong name silently fails.

### apple configuration record

- `name`, `icon`, `notes`
- `memory`, `cpu cores`
- `drives` — list of `apple drive configuration` records (`id`, `removable`, `host size`, `guest size`, `source` (file))
- `network interfaces` — list of `apple network configuration` records (`index`, `mode` ∈ `shared` | `bridged`, `address`, `host interface`)
- `serial ports` — list of `apple serial configuration` records (`index`, `interface` — only PTTY is supported on the Apple backend)
- `displays` — list of `apple display configuration` records (`id`, `dynamic resolution`)
- `directory shares` — list of `apple directory share configuration` records — only `index` and `read only` are exposed via AppleScript. **There is no `path` property** on this record; the host directory is bound through the registry, not via the configuration record. To rebind a share to a different host folder use `update registry` (see [Registry suite](#registry-suite)).

The QEMU and Apple sub-records all carry an `index` property used to identify which existing entry an `update configuration` call should replace. Omit `index` to create a new entry.

### Example: edit RAM and CPU on a stopped QEMU VM

```javascript
// JXA: osascript -l JavaScript edit-ram.js "Ubuntu" 4096 4
ObjC.import('stdlib');
const [name, memory, cores] = $.NSProcessInfo.processInfo.arguments.js
    .slice(4).map(a => a.js);
const utm = Application("UTM");
const vm = utm.virtualMachines.byName(name);
if (vm.status() !== "stopped") { console.log("VM must be stopped"); $.exit(1); }
const cfg = vm.configuration();
cfg.memory = parseInt(memory, 10);
cfg["cpu cores"] = parseInt(cores, 10);
utm.updateConfiguration(vm, { with: cfg });
```

`update configuration` accepts a partial record — fields you omit are left unchanged, but in practice it is safest to read the current configuration, mutate, and write back.

## Guest agent suite

Requires the QEMU guest agent inside the guest. Apple-backend VMs do not respond to these commands.

### query ip

```applescript
tell application "UTM" to query ip for virtual machine named "Ubuntu"
-- returns a list of text, IPv4 addresses first
```

### execute / get result

```applescript
tell application "UTM"
    set vm to virtual machine named "Ubuntu"
    set proc to execute vm at "/bin/sh" with arguments {"-c", "uname -a"} ¬
        with environment {"LANG=C"} ¬
        output capturing true
    repeat
        set r to get result of proc
        if exited of r then
            return (output text of r)
        end if
        delay 0.5
    end repeat
end tell
```

`execute` parameters:

| name | type | notes |
| --- | --- | --- |
| `at` | text | absolute path or PATH-resolvable executable |
| `with arguments` | list of text | optional |
| `with environment` | list of `NAME=VALUE` text | optional |
| `using input` | text | optional stdin |
| `base64 encoding` | boolean | if true, `using input` is base64 of binary stdin |
| `output capturing` | boolean | required for `output text`/`error text` to be filled |

`get result` returns an `execute result` record:

- `exited` — bool
- `exit code` — int
- `signal code` — int (0 if normal exit)
- `output text`, `error text` — captured stdout/stderr (if `output capturing`)
- `output data`, `error data` — base64 of stdout/stderr (use for binary)

### Guest file I/O

`utmctl file push/pull` cover the common cases. AppleScript exposes the underlying primitives if you need offsets, partial reads, or persistent handles:

```applescript
tell application "UTM"
    set f to open file for virtual machine named "Ubuntu" at "/var/log/syslog" for reading
    set chunk to read f for length 8192 base64 encoding false closing false
    close f
end tell
```

Commands: `open file`, `read`, `write`, `pull`, `push`, `close`. `read`/`write` accept `at offset N` plus `from start position | current position | end position`. The `read` `for length` limit is 48 MB.

`open file for` modes: `reading` (must exist), `writing` (truncate or create), `appending` (create if missing). `updating: true` allows read+write.

## Input automation

QEMU only. The Apple backend silently ignores these (they error with "not supported" on the underlying VM).

### input scan code

Send raw PC-AT scan codes (8-bit, with optional `0xE0xx` extended). UTM toggles the high `0x80` bit for key release internally if needed.

```applescript
tell application "UTM" to input scan code virtual machine named "Ubuntu" codes {28}
-- 28 (0x1C) = Enter
```

### input keystroke

ASCII string + optional modifiers. Modifiers are held for the entire string.

```applescript
tell application "UTM" to input keystroke virtual machine named "Ubuntu" text "ls -la" with modifiers {control}
```

Modifier keys: `caps lock`, `shift`, `control`, `option`, `command`, `escape`.

For a "press Ctrl-Alt-Del" you cannot use `input keystroke` (it sends ASCII text); use scan codes:

```applescript
tell application "UTM" to input scan code virtual machine named "Win" codes ¬
    {29, 56, 57427, 57427+128, 56+128, 29+128}  -- C-A-Del down, Del up, Alt up, Ctrl up
```

### input mouse click

Absolute coordinates inside the SPICE display.

```applescript
tell application "UTM" to input mouse click virtual machine named "Ubuntu" ¬
    at {640, 480} to 1 with mouse button left
```

`to N` selects monitor index (1-based); button is `left`, `right`, or `middle`.

## USB suite

QEMU only. The application has a `usb devices` collection of all host devices visible to UTM.

```applescript
tell application "UTM"
    set d to first usb device whose name contains "YubiKey"
    connect d to virtual machine named "Win"
    -- … later …
    disconnect d
end tell
```

USB device properties (read-only): `id` (location), `name`, `manufacturer name`, `product name`, `vendor id`, `product id`. `vendor id`/`product id` are integers — convert to hex if you compare against `VID:PID` strings.

## Registry suite

```applescript
registry of <vm>                          -- read: returns a list of file specifiers
update registry <vm> with <list-of-files> -- write: replaces ALL shared-directory entries
```

The scripting surface for the registry is narrow. Per `UTMScriptingRegistryEntryImpl.swift`, it exposes only the VM's **shared directories** — `serializeRegistry()` returns `registry.sharedDirectories.compactMap { $0.url }`, and `update registry` calls `removeAllSharedDirectories()` and re-adds the supplied URLs as bookmarks.

Implications:

- It cannot swap a removable-media ISO at runtime via this command.
- It cannot edit external drive bookmarks.
- `update registry` is **all-or-nothing** for shared dirs — pass the complete list, not a delta. To remove all shares, pass `{}`.

```applescript
tell application "UTM"
    set vm to virtual machine named "Ubuntu"
    -- read existing shares
    set shares to registry of vm
    -- swap one and rebind everything in one shot
    set newShares to {file "/Users/me/projects", file "/Users/me/data"}
    update registry vm with newShares
end tell
```

For other "registry-like" operations (changing the source file behind a removable drive, etc.) there is no scripting hook — they require the GUI.

## Serial ports

```applescript
serial ports of <vm>     -- list of "serial port" records
```

Each port has `id` (index), `interface` (`ptty` | `tcp` | `unavailable`), `address`, and `port`.

```applescript
tell application "UTM"
    repeat with p in serial ports of virtual machine named "Ubuntu"
        log (interface of p as text) & " " & (address of p) & ":" & (port of p as text)
    end repeat
end tell
```

For `ptty`, `address` is the host pseudo-tty path — open with `screen /dev/ttysNNN` or `cu -l`. For `tcp`, connect with `nc <address> <port>`.

## JXA recipe collection

JXA is reachable from the shell as `osascript -l JavaScript -e '<code>'` or from a `.scpt`/`.js` file. Boilerplate:

```javascript
const utm = Application("UTM");
utm.includeStandardAdditions = true;
```

### List VMs as JSON

```javascript
const utm = Application("UTM");
JSON.stringify(utm.virtualMachines().map(vm => ({
    id: vm.id(),
    name: vm.name(),
    backend: vm.backend(),
    status: vm.status(),
})), null, 2);
```

Run with: `osascript -l JavaScript list.js`.

### Start, wait for IP, run a command

```javascript
const utm = Application("UTM");
const vm = utm.virtualMachines.byName("Ubuntu");

if (vm.status() !== "started") utm.start(vm);

let ip;
for (let i = 0; i < 60 && !ip; i++) {
    try {
        const ips = utm.queryIp(vm);          // throws if agent not ready
        ip = ips.find(a => a.includes("."));
    } catch (_) { delay(2); }
}
if (!ip) throw new Error("guest never came up");

const proc = utm.execute(vm, {
    at: "/bin/sh",
    withArguments: ["-c", "uname -srm && hostname"],
    outputCapturing: true,
});

let r;
do { delay(0.3); r = utm.getResult(proc); } while (!r.exited);
console.log(`exit=${r.exitCode}\n${r.outputText}`);
```

JXA snake-cases the AppleScript parameter names: `with arguments` → `withArguments`, `output capturing` → `outputCapturing`, `with modifiers` → `withModifiers`, `mouse button` → `mouseButton`.

### Type into the login screen

```javascript
const utm = Application("UTM");
const vm = utm.virtualMachines.byName("Ubuntu");
utm.inputKeystroke(vm, { text: "myuser" });
utm.inputScanCode(vm, { codes: [28] });           // Enter
utm.inputKeystroke(vm, { text: "secretpw" });
utm.inputScanCode(vm, { codes: [28] });
```

### Create a stripped-down ARM Linux VM

```javascript
const utm = Application("UTM");
utm.make({
    new: "virtual machine",
    withProperties: {
        backend: "qemu",
        configuration: {
            name: "tiny-arm",
            architecture: "aarch64",
            machine: "virt",
            memory: 1024,
            "cpu cores": 2,
            uefi: true,
            hypervisor: true,
        },
    },
});
```

You then need to attach a disk and an installer ISO, which is much easier from the GUI wizard — see [workflows.md](workflows.md).

### Take a snapshot via guest-side fsfreeze (no UTM-native snapshot CLI)

UTM does not expose a "snapshot" verb in the dictionary. To create application-consistent backups, freeze the guest filesystem first, then `cp -R` the bundle:

```bash
osascript -l JavaScript <<'JS'
const utm = Application("UTM");
const vm = utm.virtualMachines.byName("Ubuntu");
utm.execute(vm, { at: "/usr/sbin/fsfreeze", withArguments: ["-f", "/"] });
JS
cp -Rc "$HOME/Library/Containers/com.utmapp.UTM/Data/Documents/Ubuntu.utm" /Volumes/backup/
osascript -l JavaScript -e '
    const utm = Application("UTM");
    utm.execute(utm.virtualMachines.byName("Ubuntu"),
                { at: "/usr/sbin/fsfreeze", withArguments: ["-u", "/"] });'
```

(Use APFS `cp -c` to clone reflinks — the bundle can be tens of GB.)
