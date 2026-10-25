# Phantom Macro Reference

A **macro** is a plain-text script — think a `.bat` file — that runs a sequence of phantom commands with variables, conditionals, loops, functions, and error handling. This is the complete reference; run **`macref`** in phantom any time for the same cheat sheet in-game.

---

## Table of contents

- [Quick reference](#quick-reference)
- [Where macros live](#where-macros-live)
- [Creating & running](#creating--running)
- [Arguments](#arguments)
- [Comments](#comments)
- [Variables & clips](#variables--clips)
- [Inline flow (single line)](#inline-flow-single-line)
- [Conditionals](#conditionals)
  - [Condition keywords](#condition-keywords)
  - [Comparisons & logic](#comparisons--logic)
  - [Stack conditions](#stack-conditions)
- [Loops](#loops)
- [`foreach` vs `for`](#foreach-vs-for)
- [Switch](#switch)
- [Error handling](#error-handling)
- [Functions](#functions)
- [Menus](#menus)
- [Misc](#misc)
- [Full example](#full-example)

---

## Quick reference

| Area | Syntax |
|---|---|
| **Run** | `mac <name> [args]` · `mac list` · `mac show <name>` · `mac trace <name> [args]` |
| **Variables** | `clip @var value` · `@var` (expand) · `$1 … $9` (macro args) |
| **Flow** | `cmd1 \| cmd2` (pipe, stop on fail) · `cmd1 && cmd2` (and) · `cmd1 ; cmd2` (always both) |
| **If** | `if <cond> then <cmd>` · block: `if` / `elif` / `else` / `endif` |
| **Loops** | `while <cond> [max N]` · `until <cond> [max N]` · `repeat N` · `for @v in @list` · `for @v in range(N,M)` |
| **Loop ends** | `endwhile` · `enduntil` · `endrepeat` · `endfor` · `break` · `continue` |
| **Switch** | `switch @var` / `case <val>` / `default` / `endswitch` |
| **Errors** | `try` / `catch [@err]` / `endtry` · `_throw <msg>` |
| **Functions** | `func <name>(p1,p2,…)` / `endfunc` / `_return <val>` · `call <name> [args]` |
| **Menus** | `menu <title>` / `<label> \| <command>` / `endmenu` |
| **Misc** | `sleep N` · `pause` |

---

## Where macros live

Macros are files in `/root/ai/mac/` on the machine phantom is running on. The filename (minus `.txt`) is the name you invoke:

```
/root/ai/mac/hunt.txt      ->  mac hunt
/root/ai/mac/rootall.txt   ->  mac rootall
```

The `.txt` extension is a convention, not a requirement — `mac`/`call` will also find a file with no extension if `<name>.txt` doesn't exist.

---

## Creating & running

1. Open the in-game code editor and create a text file at `/root/ai/mac/<name>.txt`.
2. Write one phantom command per line (syntax below).
3. Save — **no build step**; macros are read and compiled fresh every run.

```
mac list                  # every macro currently in /root/ai/mac/
mac show <name>           # preview a macro's lines without running it
mac <name> [args]         # run it
mac trace <name> [args]   # run it, printing each COMPILED line right before it executes
```

**`mac trace`** is for debugging. It prints the *compiled* line — including the internal sentinels a `for`/`if`/`while` compiles down to (e.g. `__for__…`) — so you can confirm a loop is iterating the values you expect, or a conditional is taking the branch you expect.

---

## Arguments

Anything after the macro name is available as `$1` … `$9`:

```
// /root/ai/mac/hunt.txt
// usage: mac hunt <port>
netmap -l
scan pt $1
grab pt $1
```

```
mac hunt 80
```

If a macro references `$4` but you passed only 3 args, phantom prints a warning and leaves it blank rather than failing silently.

**Piped input becomes `$1`.** `randip | mac hunt` hands the generated IP in as the macro's first arg — no `@clip` round-trip needed. Args you also type shift down to `$2`, `$3`, …:

```
randip | mac hunt 80
// $1 = piped ip, $2 = "80"
```

---

## Comments

A line starting with `//` is ignored entirely — not even shown to the parser.

---

## Variables & clips

| Syntax | Meaning |
|---|---|
| `clip @var value` | set a clip variable |
| `@var` | expand the clip inline in a command |
| `@var.len` | expands to the item count (comma-list) or character length |
| `$1 … $9` | macro arguments (see above) |

Clips persist across macros and at the interactive prompt. A clip holding a comma-separated string is a **list** to loops and `foreach`. A clip captured from the stack (`stack | clip @b`) is an **object-list clip** holding live shells/computers (see [`foreach` vs `for`](#foreach-vs-for)).

---

## Inline flow (single line)

No block needed:

| Syntax | Meaning |
|---|---|
| `cmd1 \| cmd2` | **pipe** — stop the line if `cmd1` fails (and pass its result on) |
| `cmd1 && cmd2` | **AND** — stop if `cmd1` fails |
| `cmd1 ; cmd2` | **always** run both, regardless of failure |

---

## Conditionals

One-liner:
```
if <cond> then <cmd>
if <cond> | <cmd>          // same thing, | instead of then
```

Block form:
```
if <cond>
    ...
elif <cond>
    ...
else
    ...
endif
```

### Condition keywords

| Condition | True when |
|---|---|
| `root` | holding a shell **and** it has root |
| `guest` | holding a shell and it does **not** have root |
| `shell` | held object is a shell |
| `computer` | held object is a computer |
| `object` | anything at all is held |
| `stack` | the object stack has ≥ 1 entry |
| `target` (or `pt`) | a public target is set |
| `failed` (or `pipefailed`) | the last pipeline step failed |
| `@var` | the clip is set and non-empty |
| `true` / `false` | constants |

Prefix any condition with `not` (or `no`) to negate it.

### Comparisons & logic

Conditions support comparisons: `==`, `!=`, `<`, `>`, `<=`, `>=`.

```
stack >= 3
shell == root
object == null
@count == 5
```

Combine with `and` / `or` — **no parentheses**, evaluated left to right by whichever operator appears first:

```
if root and stack >= 1 then autoharvest
```

### Stack conditions

Pre-built checks across every stack object (all **false on an empty stack** — no vacuous truth):

| Condition | True when |
|---|---|
| `stackanyroot` | at least one stack object currently has root |
| `stackallroot` | every stack object has root |
| `stackanyguest` | at least one stack object does **not** have root |
| `stackallguest` | none of the stack objects have root |
| `stackanycomputer` | at least one stack object is a `computer` |
| `stackallcomputer` | every stack object is a `computer` |
| `stackanyshell` | at least one stack object is a `shell` |
| `stackallshell` | every stack object is a `shell` |

**`stackhas <condition>`** is the general boolean form — same condition matching as `stackpick` (`computer`, `shell == root`, `and`/`or` chains, anything `EvalCondition` supports), but it **never touches the held object**, win or lose:

```
if stackhas computer and shell == root then echo "a rooted computer is on the stack"
// held object untouched — use stackpick if you actually want to grab the match
```

> The `stackany*`/`stackall*` keywords are just `stackhas` pre-built for common checks. Use the plain keyword when it fits, `stackhas` when you need something custom.

**Comparing against a stack snapshot.** `stack | clip @a` captures the live objects on the stack into an object-list clip.

- `stack == @a` / `!=` / `<` / `>` / `<=` / `>=` compare the **current** stack's *count* against how many objects `@a` captured — size only (two different targets swapped in place still count as "unchanged").
- `stackchanged @a` (or `not stackchanged @a`) compares the **actual objects** position by position — true if anything differs, false only if it's the exact same objects in the exact same order.
- A clip that was never taken counts as "changed" for `stackchanged`, and as `0` for the count comparison.

```
stack | clip @a
autoroot -l
if stackchanged @a then echo "stack composition changed"
if stack != @a then echo "stack size changed (could be reordered though)"
```

---

## Loops

```
while <cond> [max N]
    ...
endwhile

until <cond> [max N]        // inverse of while
    ...
enduntil

repeat N
    ...
endrepeat

for @var in @list           // iterate a comma-separated clip; current item is @var
    ...
endfor

for @var in range(5,10)      // iterate 5..10
    ...
endfor
```

- **`max N`** on `while`/`until` caps iterations as a safety net (default cap is effectively unlimited).
- **`break` / `continue`** work inside any loop.
- **A failed pipe stage inside a loop body skips to the next iteration, not the whole macro.** If `scan @x 80 | grab @x 80` fails partway (e.g. `scan` finds nothing on that item), the loop moves to the next item instead of aborting — for `for` / `while` / `until` / `repeat` alike. This includes a failure on the loop's *last* item: whatever you write after `endfor`/`endwhile` still runs.

---

## `foreach` vs `for`

They are **different constructs**:

- **`for @var in @list … endfor`** is macro-only and lets you name the loop variable.
- **`foreach @list | cmd @each`** is a prompt-level construct (works outside macros too, straight at the phantom prompt). It queues one pipeline run per list item, substituting `@each`.

**Iterating live objects.** If `@list` is an *object-list clip* (from `stack | clip @b`) rather than a string clip, `for @x in @b … endfor` can't substitute a live shell/computer as text — so instead it sets the active `object` to the current item for each iteration's body. Write the body like you would for `stackfor`: reference the held target implicitly (`harvest`, `scan pt 80`), **not** via `@x`.

```
// harvest every object wstack found
stack | clip @b
for @x in @b
    harvest
endfor
```

---

## Switch

```
switch @var
case value1
    ...
case value2
    ...
default
    ...
endswitch
```

---

## Error handling

```
try
    ...
    _throw "something went wrong"
catch @err
    echo @err
endtry
```

`catch` without a variable (`catch` alone) still catches — it just doesn't capture the message.

---

## Functions

Inline functions, defined and callable within the same macro (or from another macro — `call` looks up both):

```
func escalate_and_grab(ip, port)
    scan $1 $2
    grab $1 $2
    if root then echo "got root on $1"
endfunc

call escalate_and_grab 1.2.3.4 80
```

- **`_return <value>`** inside a func sets the value `call` / `macall` gets back.
- **`call` / `macall`** also run a *file* macro by name (not just an inline `func`) — same command either way: it checks inline funcs first, then `/root/ai/mac/`.

---

## Menus

Interactive choice prompt — pauses the macro until you pick one:

```
menu Pick a target
1 | scan pt 80
2 | scan pt 8080
q | exit
endmenu
```

Each line is `<label> | <command>` — typing the label at the prompt runs that command and resumes the macro.

---

## Misc

```
sleep 5     // wait 5 seconds
pause       // wait for Enter
```

---

## Full example

```
// /root/ai/mac/rootall.txt
// usage: mac rootall
echo "mapping local network..."
netmap -l
lanenum -l @targets

for @ip in @targets
    scan @ip 80
    if root then
        echo "rooted @ip"
    else
        scan @ip 8080
        if root then echo "rooted @ip via 8080"
    endif
endfor

echo "done"
```
