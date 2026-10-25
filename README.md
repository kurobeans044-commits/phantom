# phantom

A GreyScript hacking toolkit for [Grey Hack](https://store.steampowered.com/app/605230/Grey_Hack/) — a custom shell that layers **piping, variables, aliases, macros, an object stack, and a persistent exploit database** on top of the game's scripting API, plus a large set of recon/exploitation commands for local and remote targets.

> Every command is self-registering: a topic file (`ops.src`, `hack.src`, …) defines `CmdFoo = function(inp) … end function` and immediately follows it with `CommandTable["foo"] = @CmdFoo`. Adding or finding a command is a one-file, one-location affair — no separate manifest to keep in sync.

---

## Contents

- [Requirements](#requirements)
- [Installation](#installation)
- [First run](#first-run)
- [Core concepts](#core-concepts)
- [Jumping, libs & sessions](#jumping-libs--sessions)
- [Command reference](#command-reference)
  - [Local](#local)
  - [Network / recon](#network--recon)
  - [Hacking](#hacking)
  - [Remote](#remote-act-on-the-held-object)
  - [Stack](#stack)
  - [Variables & scripting](#variables--scripting)
  - [Misc](#misc)
- [Plugin modules](#plugin-modules)
- [Payload scripts](#payload-scripts)
- [Persisted files](#persisted-files)
- [Development notes](#development-notes)

---

## Requirements

- Grey Hack, with `metaxploit.so` and `crypto.so` available (at `/lib/` or alongside phantom's files).
- Disk space for phantom's `.src` files plus whatever `db.txt` / `deadlibs.txt` / `aliases.txt` grow to over a session.

---

## Installation

GreyScript's `import_code` needs a **literal** path known when the script loads — it can't take a computed or relative path. So **phantom must live at `/root/ai/pt/`**, the path hardcoded in `phantom.src`:

```greyscript
import_code("/root/ai/pt/macro.src")
import_code("/root/ai/pt/hack.src")
import_code("/root/ai/pt/ops.src")
import_code("/root/ai/pt/remote.src")
import_code("/root/ai/pt/chainsaw.src")
import_code("/root/ai/pt/stack.src")
import_code("/root/ai/pt/game.src")
import_code("/root/ai/pt/rt.src")
```

Installing elsewhere? Edit those eight lines to match.

> Some comments elsewhere say `/root/ai/tp/` instead of `/root/ai/pt/` in `build` suggestions — that's just stale comment text, not a requirement. `build` uses whatever path you type, so use your actual install path.

**Steps:**

1. Copy every root `.src` file (`phantom.src`, `ops.src`, `hack.src`, `remote.src`, `chainsaw.src`, `stack.src`, `macro.src`, `game.src`, `rt.src`) into `/root/ai/pt/`.
2. Copy the `data/` folder (`pregens.src`, `samples.src`) alongside them — `chainsaw.src` / `rt.src` load these as password wordlists, falling back to a small built-in list if missing.
3. Make sure `metaxploit.so` and `crypto.so` are reachable (in `/lib/` or copied next to `phantom.src`).
4. Build it: `build /root/ai/pt/phantom.src /root/ai/pt` → produces the runnable `phantom` binary.
5. Copy the `payloads/` folder too (`getlogs.src`, `libscan.src`, `netmap.src`, `slb.src`, `wsx.src`). These aren't imported at startup — they're uploaded to targets on demand and only need building the first time each is used (phantom prints the exact `build` command if it can't find a compiled copy).
6. *Optional:* copy `modules/` and build `modules/hello.src` to try the plugin system (see [Plugin modules](#plugin-modules)). Skip it entirely otherwise — phantom runs fine with no `modules/` folder.

> **Heritage:** `hack.src` absorbed the former `core.src`; `macro.src` absorbed `misc.src`; `stack.src` absorbed `vars.src`; `recon.src` was dropped (its content was already duplicated in `remote.src`).

---

## First run

```
./phantom
```

On startup phantom loads `metaxploit.so` / `crypto.so`, auto-creates/loads `db.txt`, `deadlibs.txt`, and `aliases.txt` (if present), then drops you at a prompt.

Type **`help`** for the command list — `help -a` for everything, or `-l` / `-n` / `-h` / `-r` for local / network / hacking / remote subsets.

**Headless:** `phantom --rshell [port]` installs and starts an rshell daemon on the local machine instead of opening the interactive shell (default port 1222).

---

## Core concepts

### Piping

```
scan pt 0 | grab pt 0      # pipe a command's result into the next
cmd |> other               # queue `other` as a separate stage, after cmd fully finishes
```

`|>` runs its stage once the first command completes — *including* anything it queued via macros/foreach.

### Clips (`@var`)

Store a result by ending a command with a bare `@name`, then reference it later:

```
randip @ip                 # store
scan @ip 80                # use
@ip.len                    # length (or comma-count for a list)
```

- `@o key` reads any top-level custom-object index directly (not just clips).
- `@lan` is a **read-only token** (not a stored clip) that always resolves to the held `object`'s LAN ip — e.g. `uselib -l @lan` in a macro instead of `ifconfig`-ing and clipping it. Warns and leaves the token untouched if nothing's held.
- Clips persist across restarts via the game's custom object.

### Object-list clips

`stack | clip @b` captures the **live** shell/computer objects on the stack into `@b` (a separate store from string clips — a shell reference can't be joined into text). Iterate them:

```
foreach @b | <cmd>         # at the prompt; sets active object to each in turn
for @x in @b … endfor      # in a macro
```

Great after `wstack` populates the stack with several targets at once.

### Local vs. remote state

- `lObject` / `myShell` / `myComp` — where you stand **locally**.
- `object` / `remotePath` — a held **remote** shell/computer.

Most commands are either local (`ls`, `cd`, `ps`) or remote (`rls`, `rcd`, `rps`); the remote ones act on whatever's held in `object`.

### The stack

`push` / `pop` / `stack` / `swap` juggle multiple shell/computer objects at once — handy when a scan turns up several targets and you don't want to lose the earlier ones.

### Aliases

```
alias name cmd args...     # define a shortcut
alias -d name              # remove it
```

Persisted to `aliases.txt`.

### Macros

`.bat`-style scripts under `mac`'s folder (`/root/ai/mac/`), supporting `if` / `while` / `for` / `switch` / `try`/`catch` / inline `func`. Run with `mac <name>`.

- `mac trace <name>` runs it but prints each **compiled** line before it executes (including internal control-flow sentinels) — for finding exactly where a misbehaving macro goes wrong.
- Piping into `mac` feeds the result as `$1`: `randip | mac t1` hands the ip in as the first arg, shifting typed args to `$2`, `$3`, …

See **[MACROS.md](MACROS.md)** for the full syntax guide.

### The exploit database (`db.txt`)

Every overflow attempt — success or failure, from any command — is recorded per exact **lib + version + mem + val**. A repeat encounter with the same lib+version skips straight to known-good candidates instead of re-scanning. `edb` inspects/manages it.

---

## Jumping, libs & sessions

### jump / jumpbin

`jump` copies phantom onto a held target and launches it there — an independent phantom session running physically on that machine. Once there, `scanlan` / `netmap` / `lanenum` all accept `-l` to work on that machine's own network directly.

| Command | Does |
|---|---|
| `jump` | upload the whole toolkit folder + launch |
| `jump -b` / `--bin` | upload just the compiled `phantom` binary (quicker round trip) |
| `jumpbin` | same as `jump -b`, no flag parsing — **more reliable**; use it if `jump -b` misbehaves |

### jump --loadlib — one-shot remote lib grab

`jump --loadlib <name>` (or `jumpbin --loadlib <name>`) skips the interactive nested session: the target auto-loads that lib **from its own `/lib`** and exits immediately, handing the loaded lib back through `gco.local`. `--loadlib` implies `-b`/`--bin`. Chain with `|>` for a full one-shot:

```
jump --loadlib libhttp.so |> uselib -l 10.0.15.5 -c
```

> Careful: this loads from the **target's** `/lib`, not yours. (For the opposite direction — your lib onto the target — use `loadmx`/`loadlib` locally *before* jumping.)

### Carrying libs across a jump

Whatever's in `globals.activeLib` / `globals.activeMx` when you `jump` **auto-carries in**, and **auto-returns** when you `exit`/`quit` — no `leave`/`claim` needed for that round trip (carried via `gco.local`).

| Command | Does |
|---|---|
| `loadlib <name>` | load a lib from the **current** machine's `/lib` into `globals.activeLib` (`loadlib -c` clears) |
| `loadmx [path]` | same for `globals.activeMx` (defaults to phantom's own `metaxploit.so`; `-c` clears) |
| `libvers` | list every lib in the current machine's `/lib` with its version |
| `activelib` / `libdebug` | debug: inspect `globals.activeLib` / `gco.claimLib` |

> Run `loadmx` **before** `jump`/`jumpbin` to "scan with my own metaxploit instead of the target's" — the `gco.local` carry brings it along, and `uselib` prefers `globals.activeMx` the moment it's set. Loading a lib/mx *after* jumping loads it from the machine you're now standing on — be deliberate about which side you run it on.

### leave / claim — hand-off between independent sessions

For two sessions that only overlap via `gco` (e.g. one jumped, one original) — not a direct jump-in/out. Works because `get_custom_object()` is a persistent, player-scoped store shared by every phantom session you run.

| Command | Does |
|---|---|
| `leave` / `claim` | hand a held shell/computer between sessions |
| `leave -lib <name> [port]` | store a lib name (+ held shell's ip/port); `claim` loads it locally from `/lib`, else redials ip:port and `dump_lib`s it |
| `leave -mx [path]` | store a `metaxploit.so` copy into `globals.activeMx` (a target's mx can decode things yours can't) |
| `leave -result` / `claim -result` | relay **db.txt scan knowledge** for the lib in `globals.activeLib` (see below) |

`claim` alone auto-detects shell → lib → mx → result in that order — which only works cleanly with one thing pending. Use explicit `claim -shell` / `-lib` / `-mx` / `-result` to grab exactly one.

> Watch for a same-named-but-different-version local lib shadowing the intended remote one — local load is tried first and doesn't check version.

### uselib

The command that consumes `activeLib` / `activeMx`: scans with `globals.activeMx` if held (else default metaxploit), attacks `globals.activeLib`, printing which lib+version it uses.

| Flag | Effect |
|---|---|
| `-s` | recon only — decode + record candidates to `db.txt`, no attack |
| `-l <lanip>` | bounce onto a LAN device (same as a bare positional inject arg) |
| `-c` | computer-**only** (rejects a shell result); implies `-a` |
| `-a` | try every candidate, push every match to the stack (shells-only unless `-c`) |

### Relaying a lib scan between sessions

`db.txt` is a per-machine *file*, not shared via `gco`. To relay a scan from session B to session A:

```
# session B:   leave -lib <name>
# session A:   claim -lib        (loads it)
# session A:   uselib -s         (scans + records to A's db.txt)
# session A:   leave -result     (packages db.txt entries into gco.claimResult)
# session B:   claim -result     (merges them into B's db.txt)
```

Lets a short-lived jumped session lean on a long-lived install's accumulated exploit knowledge without syncing `db.txt` by hand.

---

## Command reference

Run **`help -a`** in phantom for the always-current list. Grouped summary below.

### Local

| Command | Description |
|---|---|
| `ls` `tree` `cd` `cat` | filesystem browsing |
| `cp` `mv` `rm` `mkdir` `touch` | file ops |
| `chmod` `chown` `chgrp` | permissions |
| `search` | find files by keyword |
| `ps` `kill` | processes |
| `passwd` `useradd` `userdel` | local users |
| `pwd` `whoami` `clear` | basics |
| `decipher` | crack a hash (rainbow table → crypto.decipher) |
| `cover` | clean up traces |
| `corlog` | corrupt system.log on current object |
| `escalate` | escalate local privileges via lib scan |
| `terminal` | open a new local terminal |
| `randip` | generate a random IP (`randip N` for a bulk comma-list) |
| `drop` | drop the held object (`-l` local, `-r` remote) |

### Network / recon

| Command | Description |
|---|---|
| `nmap [ip] [-y]` | `-y` auto-confirms setting the scanned ip as target (implied inside a macro) |
| `ping` `nslookup` `whois` `whoismail` | lookups |
| `ifconfig` | local network info |
| `set` / `unset` | set/clear target IPs |
| `ssh [user@pass] [ip] [port]` | SSH with raw creds, or `ssh [name]` for a saved portal |
| `sshsave` / `sshlist` / `sshdel` | save/list/remove **your own** logins (→ `sshportals.txt`) |
| `scp` | upload/download a file over the held shell |
| `routerdata` | router info (ESSID/BSSID, ports, LAN devices) |
| `proxy` / `proxystatus` / `proxyclear` | anonymizing proxy chain |
| `netmap [-l]` | LAN recon only, no attacking (`-l` local) |
| `hacklan` | bounce to a LAN device |
| `lanenum [-l]` | list LAN IPs (`-l` local) |
| `ipenum` | find LAN IPs with a given port open |
| `ports <ip>[,ip2…] [ports] [-r]` | check a port list against one/many IPs; `-r` = metaxploit-free router scan (public IPs) |

### Hacking

| Command | Description |
|---|---|
| `edb` | inspect/manage the exploit database |
| `scan [ip] [port] [lanip] -lib <name> [mem val]` | scan a lib; `-lib` uses db-known-first logic; add `mem val` to force a pair |
| `scanlan [ip] [port] [-c] [-l]` | scan/attack whatever's bound to an ip:port (`-l` local, `-c` accept computers) |
| `scanlib <name> [lanip] [-c]` | scan/attack a **named** lib on the held shell's target; loaded lib lands in `globals.activeLib` |
| `grab [ip] [port] [mem val] [inject] [-c] [-push]` | overflow to grab a shell/computer |
| `exploit [ip] [port] [-c]` | interactive candidate picker |
| `report` | exploit report — see note below |
| `getroot` | escalate using a known password |
| `autoroot [-l]` | ChainSaw password cracker (`-l` local) |
| `ar [-l]` | dictionary attack via rainbow table (`-l` local) |
| `enum [-l] [-r]` | full filesystem dump |
| `harvest` | crack Mail.txt / Bank.txt / `/etc/passwd` (`harvest \| clip @b`) |
| `peek [--stack]` | like harvest but **no cracking** — raw `user:hash` pairs (e.g. `peek \| clip @a \| decipher`) |
| `siphon [-a/-b/-m/-p/-l/-n]` | network-wide harvest |
| `wifi` | scan + crack nearby WiFi |
| `sniff [-e]` | passively capture SSH/FTP creds on the current device — **closes the session when it catches one** |
| `smtplist <ip> <port>` | list every username+email on a mail service (reduces attack to password-only) |
| `wstack [-l] [libname]` | bounce the LAN, push **root computers** to the stack (see note) |
| `wstackall [-l] [libname]` | same, but also keeps guest-level bounces |
| `cascade [libname]` | run `wstack`'s remote mode from every shell on the stack |
| `pwnlan` | exploit all LAN devices |
| `crack [hash]` | crack a single hash |
| `buildrt [order] [min] [max]` | build the rainbow table in memory |
| `rtgen` | build + write the rainbow table to disk (learns slow-deciphered hashes into `learned.txt`) |
| `plant [lib…] [-d dest] [-m [path]] [--stack] [-ls]` | upload weaklib(s) from `/root/InsecureLibs` (+ always metaxploit.so) |
| `getlogs <sitelistfile> [destfolder]` | bulk site-list auto-exploiter → downloads `/var/system.log` from each rooted host |
| `rshell` | reverse-shell server — see note |
| `chainsaw run\|load\|deploy` | Markov/pregen password cracker |
| `upgrade [-u]` / `updatelib` | check/install local lib updates |
| `reset` | clear in-session `scanlan`/`report` caches (not the rainbow table/clips/aliases) |

<details>
<summary><b>report</b> — details</summary>

- `report [ip] [port]` — live-tests **every** candidate for the bound lib+version (not just the first hit like scan/grab), printing `Hooked: <user> <type>` per working one and logging all to db.txt. Known candidates skip the scan/decode step; only a genuinely new lib+version pays for a full scan.
- `report` or `report [libname] [version]` — prints from db.txt with **zero** live scanning (most-recent or a named lib+version).
- Add `-v` to any form for a `MEMORY VULNER TYPE USER` aligned table.
</details>

<details>
<summary><b>wstack</b> — details</summary>

- **No `-l`:** uploads + launches on the held shell so you don't have to jump first. This is more than convenience — `get_router(ip)`/`devices_lan_ip` from *outside* a network returns a far more limited view (one address per subnet) than running *inside* it.
- **`-l`:** run inline instead (you must already be on that LAN).
- No libname: uses `globals.activeLib` if claimed/loaded, else `init.so`/`libhttp.so`. Whatever lib actually gets used lands back in `globals.activeLib`.
- Tries every candidate per device, pushing a device only once a **root** computer bounce lands (a guest-only hit is discarded). `wstackall` keeps guest-level bounces instead.
</details>

<details>
<summary><b>rshell</b> — details</summary>

- `rshell init [-l] [port]` — install the server on the held remote shell, or `-l` to install on wherever phantom is physically running (e.g. right after `jump`).
- `rshell all [cmd…]` — catch connections, then run `cmd` against **every** shell that connected. `cmd` can be a `;`-separated pipeline (e.g. `grab ip port ; push`), stop-on-fail per shell. `;` (not `|`) is deliberate, to dodge a GreyScript `split()` quirk. No `@var`/`@o` substitution inside it.
</details>

### Remote (act on the held object)

| Command | Description |
|---|---|
| `rls` `rtree` `rcd` `rcat` | remote browsing |
| `rcp` `rmv` `rrm` `rmkdir` `rtouch` | remote file ops |
| `rchmod` `rchown` `rchgrp` | remote permissions |
| `rsearch` `rgrep` `rfind` | remote search |
| `rps` `rkill` | remote processes |
| `rpasswd` `ruseradd` `ruserdel` | remote users |
| `rwhoami` | current user on the held remote shell |
| `rcorlog [-y]` | corrupt the remote system.log (`-y` auto-confirms) |
| `rterminal` | open a terminal on the remote machine |
| `rdecipher [-h user:hash] [-f /path]` | crack hashes on the held remote object |
| `profile [-w] [-p <path>]` | plain-text recon profile of the held target (`-w` saves to `reports/<hostname>.txt`) |
| `term` `mail` `b` `file` | launch Terminal/Mail/Browser/FileExplorer **locally** |
| `rterm` `rb` `rfile` | launch Terminal/Browser/FileExplorer on the **held remote shell** |

### Stack

| Command | Description |
|---|---|
| `push` `pop` `stack` `swap` | juggle held objects |
| `stackpick <condition>` | jump to the first stack object matching a [macro condition](MACROS.md) (e.g. `stackpick shell == root`) |
| `stackfor <condition> \| <command>` | run `<command>` against **every** matching object |
| `stackrun <mode>` | run a file-op across the stack (`stackrun help`); includes `find` / `harvest` / `peek` / `ps` |
| `stacktree` | file tree per stack object (cached in `gco.treeResults`; `treelen`/`treeclear` inspect/clear) |
| `stacklog` `stackgrep` `stackfind` | log / search across the stack |
| `astack` | dictionary attack across the stack |
| `autoharvest` | auto proxy-hop + harvest |
| `harvestgrep` `psgrep` `treegrep` | grep harvested creds / process lists / cached trees |

### Variables & scripting

| Command | Description |
|---|---|
| `clip` | set/list/clear clip variables |
| `inc` `dec` `math` | arithmetic on clips |
| `alias` | define/remove command aliases |
| `ask` | prompt for input into a clip |
| `echo` `log` | print / append a line to a file |
| `write <file> [-l]` | append the pipe result one-per-line (`-l` prints the file back numbered) |
| `append <file> -l <n> -t <text>` | replace one specific line (number from `write -l`) |
| `foreach @list \| cmd @each` | run a pipeline per list item |
| `mac` `call` / `macall` `exec` | run macros / functions |
| `code` | launch the in-game code editor |
| `harveststack` `peekstack` `corlogstack` | stack-wide harvest / peek / corlog |
| `plantstack` | alias for `plant --stack` |
| `loot [-c] [-p]` | `harveststack` across the stack; `-c` also corlogstack (destructive), `-p` also plantstack |
| `rlan` `rpub` | held object's LAN / public ip |
| `p` `pre` | police records / session setup |
| `sleep N` `pause` | wait N seconds / wait for Enter |
| `exit` / `quit` | leave phantom (carries activeLib/activeMx home from a nested session) |
| `imap` | target info |

### Misc

| Command | Description |
|---|---|
| `game` / `game story` | **BREACH** — a hacking minigame (combat or branching story). `!quit` to exit. Purely for fun, touches no real state. |
| `modules` | list loaded plugin modules and their command counts |
| `monitor` / `top` `[-i secs] [-n count]` | htop-style open-port + process dashboard (held object, else local). Process/port state only — GreyScript exposes no live CPU/RAM. |
| `./name` | launch a local binary |
| `r./name` | launch a binary on the held remote shell |

---

## Plugin modules

Optional third-party commands, loaded without touching phantom's topic files or `phantom.src`'s `import_code` list. At startup phantom looks for `/root/ai/pt/modules/*.cfg`; each names a **pre-compiled binary** that gets `launch()`'d (not `import_code`'d — `launch()` accepts a runtime-computed path, which `import_code` can't). The binary writes its exports into the custom object before it exits.

A `.cfg` is plain `key=value`:

```ini
module_name=hello
binary_path=/root/ai/pt/modules/hello
init=Init
```

`module_name` / `binary_path` are required; `init` (optional) names an export-table function run once at load — return `true` on success, or a `string` to report as a load error (phantom prints it and skips that module's commands without crashing startup).

> **`binary_path` is per-install** — same caveat as the `import_code` lines. It must match wherever *you* put phantom. If it's wrong, the loader skips that module with a "binary not found" warning (no crash) and it never shows in `modules`.

A module's source, built to the binary path, ends with:

```greyscript
CmdHello = function(inp)
    print("hello from a loaded module!")
end function

Init = function()
    return true
end function

gco = get_custom_object
gco.modules["hello"] = {"commands": {"hello": @CmdHello}, "Init": @Init}
```

**Two scopes matter here:**
- The module's **top-level script** (runs once, standalone, when `launch()`'d to register exports) has **no** phantom globals — only what `get_custom_object` gives it.
- Its **command functions** (under `commands`) get merged into `CommandTable` and, once invoked, run with full native access — `object`, `myShell`, `myComp`, `c`, `activeLib`, everything.

A name collision with an existing command is skipped with a warning, not overwritten. See `modules/hello.src` + `.cfg` for the minimal example, or `modules/jumptest.src` for one that touches a held remote target.

---

## Payload scripts

Uploaded and run **on a target**, not imported at startup:

| Script | Used by | Purpose |
|---|---|---|
| `slb.src` | `scanlan` | LAN-bounce scanner — nets in from inside the segment, db-known-first then full scan |
| `libscan.src` | `scanlib` | loads a named lib directly where launched (no `net_use`), with optional LAN-ip inject |
| `wsx.src` | `wstack` remote mode | discovers the LAN itself and bounces a lib against every device; hands the loaded lib back via `gco.wsLibObj`; accepts a caller lib/mx handle (`gco.wsUseLib`/`gco.wsUseMx`) |
| `netmap.src` | `netmap` | pure recon LAN mapper — open ports, kernel/firewall info, no attacking |
| `getlogs.src` | *(reference only)* | the original standalone site-list log-puller; superseded by the `getlogs` command (which runs off `db.txt`, no upload) |

---

## Persisted files

Auto-created next to phantom's files:

| File | Purpose |
|---|---|
| `db.txt` | Exploit database — lib, version, mem, val, result type, user context for every attempt. |
| `deadlibs.txt` | Lib+version combos with zero exploitable vulns — skips the scan/decode pass next time. |
| `aliases.txt` | Persisted `alias` definitions. |
| `sshportals.txt` | Saved SSH logins (`sshsave`/`sshlist`/`sshdel`) — name, user, pass, ip, port per line. |

---

## Development notes

- **Uploaded binaries aren't cleaned up.** `wstack`/`slb`/`libscan`/`netmap` leave a compiled binary in `/home/guest` on the target for reuse — treat any pivoted host as "dirty." (`wstack -l` is the exception — inline, nothing uploaded.)
- **No true plugin folder for core files.** `import_code` can't take a computed path, so each core topic file must be a literal line in `phantom.src`. Optional commands sidestep this via `launch()` — see [Plugin modules](#plugin-modules).
- **160,000-char file cap.** Grey Hack caps a single script at 160k chars. `ops.src` grew near that, so its `r*` commands moved to `remote.src`, and `ssh`/`sshsave`/`sshlist`/`sshdel` followed. New commands should go in `chainsaw.src` (smallest topic file) unless they clearly belong with something in `ops.src`. `hack.src`/`macro.src`/`stack.src` absorbed former files, so they have less headroom than their size suggests — check `wc -c` before adding non-trivial code.
