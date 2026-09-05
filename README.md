# Warband Engine (Gloria Sinica) - Trial Limitation & Wine Fix

Binary patch documentation and executable fix for Mount & Blade Warband engine standalone titles (specifically *Gloria Sinica: Han Xiongnu Wars* v2.725) running under Wine / Proton on Linux.

## Problem Description

Standalone titles running on older Mount & Blade engine revisions (such as Engine v0.910) fail to find native Windows registry keys when executed via Wine, causing:
1. Blocking modal activation / serial key dialog at startup.
2. Hardcoded trial-mode level cap enforced at Level 8 (triggers an automated save-and-exit sequence).

## Binary Modifications

Direct PE patches applied to `HanXiongnuWars.exe` (v2.725) targeting the global activation flag conditional branches (`0x8eb5a0` in `.data`):

| # | File Offset | Original Bytes | Patched Bytes | Description |
|---|-------------|----------------|---------------|-------------|
| 1 | `0x207067` | `0f 85 80 00 00 00` | `90 e9 80 00 00 00` | Level limit check: `jnz` -> unconditional `jmp` |
| 2 | `0x207e3b` | `74 4e` | `90 90` | Trial menu branch: `jz` -> NOP sled |
| 3 | `0x1b5e93` | `0f 84 69 01 00 00` | `90 90 90 90 90 90` | Trial mode long jump -> NOP |
| 4 | `0x18b455` | `74 07` | `90 90` | Serial verification branch A -> NOP |
| 5 | `0x18deab` | `74 07` | `90 90` | Serial verification branch B -> NOP |
| 6 | `0x1bf51c` | `0f 85 85 0a 00 00` | `90 e9 85 0a 00 00` | Startup activation modal: `jnz` -> unconditional `jmp` |

## Usage

```bash
# Backup original binary
cp HanXiongnuWars.exe HanXiongnuWars.exe.bak

# Replace with patched binary
cp patch/HanXiongnuWars.exe /path/to/game/directory/HanXiongnuWars.exe

# Launch via Wine
wine HanXiongnuWars.exe
```

## Results

- Startup serial prompt bypassed directly to main menu.
- Character progression beyond level 8 fully unlocked.
- Compatible with Wine / DXVK 3.x stack on Linux.

## Methodology

See [SKILL.md](SKILL.md) for reverse-engineering workflow and branch-mapping procedure.
