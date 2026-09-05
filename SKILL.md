---
name: warband-trial-bypass
description: >-
  Bypasses serial key validation and trial-mode level caps in Mount & Blade
  Warband engine games (including standalone mods like Gloria Sinica) running
  via Wine on Linux. Locates the activation flag in the PE binary, identifies
  all conditional branches gating trial behavior, and patches them with NOP
  sleds or unconditional jumps. Also removes the serial-key popup dialog at
  startup. Backs up the original executable before any modifications.
---

# Mount & Blade: Warband Engine – Trial Mode & Serial Key Bypass (Wine/Linux)

## Overview

The Warband engine (versions 0.9xx–1.1xx) enforces a trial-mode level cap
(typically level 8) and serial-key activation dialog when no valid key is found
in the Windows registry. On Linux under Wine, the registry key lives at:

```
HKCU\Software\<GameName>Keys\serial_key
```

This skill patches the **game executable directly** to:
1. Remove the level cap enforcement (no more forced save-and-quit at level 8)
2. Remove the serial-key / activation popup dialog at startup
3. Skip all trial-mode conditional branches throughout the engine

## When to Use

- User reports a level cap (usually level 8) in a Warband-engine game under Wine
- User sees a "试用模式" / "trial mode" / "Enter your serial key" popup
- User wants to bypass serial key validation for a Warband standalone mod
- Game executable contains strings like `chkserial`, `ui_trialmode`, `serial_key`

## Prerequisites

- Python 3 (for the binary patching script)
- The game must be a 32-bit PE (x86) Warband-engine executable

## Workflow

### Phase 1: Identify the Game Binary and Activation Flag

1. **Locate the executable**:
   ```bash
   find "<game_dir>" -iname "*.exe" -maxdepth 1
   ```

2. **Verify it's a Warband engine game** by checking for signature strings:
   ```bash
   strings "<game.exe>" | grep -i "chkserial\|ui_trialmode\|serial_key"
   ```
   Expected output should include `chkserial`, `serial_key`, `ui_trialmode`,
   `ui_explanation_level_limit`, etc.

3. **Find the registry key namespace**:
   ```bash
   strings "<game.exe>" | grep "Software\\\\"
   ```
   This reveals the registry path (e.g., `Software\GloriaSinicaHanXiongnuWarsKeys`).

4. **Find the activation flag virtual address**. Search for the
   `ui_explanation_level_limit` string in the binary, compute its VA, find its
   cross-reference in `.text`, and locate the global variable address used in
   the `mov/cmp` instruction immediately preceding the conditional jump. This
   is the **activation flag** (a DWORD at a fixed VA in `.data`).

### Phase 2: Map All Trial-Mode Branch Points

Search the entire `.text` section for all references to the activation flag VA.
These fall into distinct categories:

| Pattern | Meaning | Action |
|---|---|---|
| `cmp [flag], 0` / `jz` (short `74` or near `0f 84`) | "If NOT activated, jump to trial code" | **NOP the jump** (`90 90` or `90*6`) |
| `cmp [flag], 0` / `jnz` (near `0f 85`) | "If activated, skip trial code" | **Change to unconditional jump** (`90 e9`) |
| `mov ebx, [flag]` / `test ebx,ebx` / `jnz` | "If activated, skip level-limit popup" | **Change `jnz` to unconditional jump** (`90 e9`) |
| `mov [flag], edi` | "Set activation state" | **Leave untouched** |
| `cmp [flag], X` / `sete` | "Set trial boolean from flag" | **Leave untouched** (handled by other patches) |

### Phase 3: Patch the Binary

**ALWAYS back up the original executable first:**
```bash
cp "<game.exe>" "<game.exe>.bak"
```

Use the following Python pattern to find and patch all branch points:

```python
import struct

with open("<game.exe>", "rb") as f:
    data = bytearray(f.read())

FLAG_VA = 0xXXXXXX  # The activation flag VA found in Phase 1
flag_bytes = struct.pack("<I", FLAG_VA)

# --- Patch Type A: mov reg, [flag] / test reg,reg / jnz -> unconditional jmp ---
pat_a = flag_bytes + b'\x0f\x8e' + b'....' + b'\x85' + b'??' + b'\x0f\x85'
# Search with exact bytes from disassembly, then:
# Change 0f 85 XX XX XX XX -> 90 e9 XX XX XX XX

# --- Patch Type B: cmp [flag], 0 / jz near (0f 84) -> NOP sled ---
pat_b = b'\x83\x3d' + flag_bytes + b'\x00\x0f\x84'
pos = 0
while True:
    pos = data.find(pat_b, pos)
    if pos < 0: break
    jz_offset = pos + 7
    data[jz_offset:jz_offset+6] = b'\x90' * 6
    pos += 9

# --- Patch Type C: cmp [flag], 0 / jz short (74 XX) -> NOP ---
pat_c = b'\x83\x3d' + flag_bytes + b'\x00\x74'
pos = 0
while True:
    pos = data.find(pat_c, pos)
    if pos < 0: break
    jz_offset = pos + 7
    if data[jz_offset] != 0x90:  # skip already patched
        data[jz_offset:jz_offset+2] = b'\x90\x90'
    pos += 8

# --- Patch Type D: cmp [flag], 0 / jnz near -> unconditional jmp ---
pat_d = b'\x83\x3d' + flag_bytes + b'\x00\x0f\x85'
pos = 0
while True:
    pos = data.find(pat_d, pos)
    if pos < 0: break
    jnz_offset = pos + 7
    data[jnz_offset:jnz_offset+2] = b'\x90\xe9'
    pos += 9

with open("<game.exe>", "wb") as f:
    f.write(data)
```

### Phase 4: Verify

After patching, launch the game:
```bash
cd "<game_dir>"
wine "<game.exe>"
```

Verify:
- No serial key popup appears at startup
- Character can level past 8 without forced save-and-quit
- Game otherwise runs normally (combat, menus, saving all work)

## Common Mistakes

1. **Patching `jnz` when you should NOP `jz`** — The jump direction matters.
   `jz` after `cmp [flag], 0` means "if trial mode, go to trial code" — you
   want to **NOP it** (never take the jump). `jnz` means "if activated, skip
   trial code" — you want to make it **unconditional** (always skip).

2. **Not backing up the executable** — Always `cp game.exe game.exe.bak`
   before any patching. A bad patch can make the game unlaunchable.

3. **Forgetting to search ALL references** — The activation flag is typically
   checked in 15-25 locations throughout the binary. Missing even one can
   leave a residual trial-mode behavior (like the popup still appearing).

4. **Confusing file offsets with virtual addresses** — PE sections have
   different file offsets vs. runtime VAs. Use the PE section headers to
   convert: `file_offset = raw_ptr + (target_VA - image_base - section_VA)`.
