# GD32F303 CNC Controller Firmware Analysis

**Firmware:** VGRBL-3axis-1.0.8.4-3020 (GRBL 1.1f based)
**Chip:** GD32F303 (GigaDevice ARM Cortex-M4), 256KB flash, 48KB SRAM
**Problem:** `$1` (stepper idle lock time) was set to `0xFF` — controller bricked, no UART/USB communication

---

## 1. File Identification

- **Format:** Intel HEX, base address `0x08000000` (STM32/GD32 flash base)
- **Size:** 262144 bytes (256KB) — full GD32F303 flash dump
- **Vector table:** SP=`0x20001620`, Reset=`0x08000149`
- **Code area:** `0x00000`–`0x1DFFF` (~120KB ARM Thumb-2)
- **EEPROM emulation:** `0x1F000`–`0x1F3FF` (1KB)
- **Motor config:** `0x1F400`–`0x1F7FF`
- **Erased flash:** `0x1F800`–`0x3FFFF` (all `0xFF`)

Firmware identification strings:
```
Grbl 1.1f ['$' for help]
VGRBL-3axis-1.0.8.4-3020
X_LIMIT / Y_LIMIT / Z_LIMIT / SAFETY_DOOR / Homing
```

---

## 2. EEPROM Settings Layout

Settings stored at flash address `0x0801F000`:

| Byte | Content |
|------|---------|
| 0 | EEPROM version = `0x0A` (10) |
| 1–100 | `settings_t` struct (100 bytes) |
| 101 | Checksum = `0x41` (logical OR algorithm, verified) |

### Decoded GRBL Settings

**Axis parameters (offsets 0–47, 12 × IEEE 754 float):**

| Struct offset | Hex bytes (LE) | Value | GRBL param |
|---------------|---------------|-------|------------|
| 0–3 | `00 00 48 44` | 800.0 | $100 Steps/mm X |
| 4–7 | `00 00 48 44` | 800.0 | $101 Steps/mm Y |
| 8–11 | `00 00 48 44` | 800.0 | $102 Steps/mm Z |
| 12–15 | `00 00 FA 44` | 2000.0 | $110 Max rate X (mm/min) |
| 16–19 | `00 00 FA 44` | 2000.0 | $111 Max rate Y (mm/min) |
| 20–23 | `00 00 C8 42` | 100.0 | $112 Max rate Z (mm/min) |
| 24–27 | `00 A0 8C 47` | 72000.0 | $120 Accel X (mm/sec²) |
| 28–31 | `00 A0 8C 47` | 72000.0 | $121 Accel Y (mm/sec²) |
| 32–35 | `00 A0 8C 47` | 72000.0 | $122 Accel Z (mm/sec²) |
| 36–39 | `00 00 FA C3` | -500.0 | $130 Max travel X (mm) |
| 40–43 | `00 00 FA C3` | -500.0 | $131 Max travel Y (mm) |
| 44–47 | `00 00 48 C3` | -200.0 | $132 Max travel Z (mm) |

**Scalar parameters (offsets 48–99):**

| Struct offset | Flash addr | Value | GRBL param |
|---------------|------------|-------|------------|
| 48 | `0x0801F031` | `0x0A` (10) | $0 Step pulse (µs) |
| 49 | `0x0801F032` | `0x00` | $2 Step invert mask |
| 50 | `0x0801F033` | `0x00` | $3 Dir invert mask |
| 51 | `0x0801F034` | `0x00` | Unknown |
| 52 | `0x0801F035` | `0x06` | $10 Status report |
| 53 | `0x0801F036` | `0x00` | Unknown |
| **54** | **`0x0801F037`** | **`0x00`** | **$1 Stepper idle lock (ms)** |
| 55 | `0x0801F038` | `0x01` | $6 Probe pin invert? |
| 56–59 | `0x0801F039` | 0.01 | $11 Junction deviation |
| 60–63 | `0x0801F03D` | 0.002 | $12 Arc tolerance |
| 64–67 | `0x0801F041` | 1000.0 | $24 Homing feed rate |
| 72–75 | `0x0801F049` | 24 (int) | $23 Homing dir mask? |
| 76–79 | `0x0801F04D` | 10 (int) | Unknown |
| 84–87 | `0x0801F055` | 25.0 | $25 Homing seek rate |
| 88–91 | `0x0801F059` | 500.0 | $30 Max spindle RPM |
| 92–95 | `0x0801F05D` | 250 (int) | $26 Homing debounce (ms) |
| 96–99 | `0x0801F061` | 2.0 | $27 Homing pull-off (mm) |

> **Note:** The struct field order differs from standard GRBL 1.1f. The `$1` parameter is at **offset 54** (not 51). This was confirmed by ARM disassembly — see Section 4.

---

## 3. The $1 Parameter Problem

### Standard GRBL $1 behavior:
- `$1=0` → Disable steppers immediately when idle (motors free)
- `$1=25` → 25ms delay before disabling (GRBL default)
- `$1=255` (`0xFF`) → Never disable steppers (permanent hold)

### Current state in hex file:
- `$1 = 0x00` (0) at flash address `0x0801F037`
- Settings checksum is **valid** (`0x41`)

### What likely happened:
1. User set `$1=255` via serial (`$1=255` command)
2. Motors started holding permanently from boot
3. High current draw may have caused USB power drop → no enumeration
4. UART may also be affected by power/noise issues
5. The firmware may have auto-restored defaults on a subsequent boot (if checksum validation failed), which would explain why `$1=0` (not `0xFF`) is in the dump

---

## 4. ARM Disassembly Findings

### 4.1 EEPROM Init (0x08002DA4)

The firmware copies 1024 bytes (`0x400`) from flash (`0x0801F000`) to RAM (`0x20000411`), then validates the version byte:

```arm
; Copy 1024 bytes from flash EEPROM to RAM buffer
copy_loop:
    ldr     r2, [pc, #0x34]         ; r2 = 0x0801F000 (flash base)
    ldrb    r2, [r0, r2]            ; read byte
    strb    r2, [r1], #1            ; write to RAM, r1++
    cmp.w   r0, #0x400              ; 1024 bytes
    blt     copy_loop

; Check EEPROM version
    ldr     r2, [pc, #0x20]         ; r2 = 0x0801F000
    ldrb    r2, [r2]                ; version byte
    cmp     r2, #0xa                ; expected: 10
    beq     valid                   ; OK

; Invalid version → fill RAM with 0xFF (reset all settings)
    movs    r2, #0xff
fill_loop:
    strb    r2, [r1], #1
    cmp.w   r0, #0x400
    blt     fill_loop
```

### 4.2 Stepper Idle Lock Check (0x08007F8C) — KEY FINDING

```arm
st_go_idle:
    push    {r4, lr}
    ; ... GPIO/timer setup ...

    ldr     r0, [pc, #0x5c]         ; r0 = 0x20001054 (settings struct in RAM)
    ldrb.w  r0, [r0, #0x36]         ; r0 = settings->stepper_idle_lock_time
                                     ;       OFFSET 0x36 = 54 DECIMAL
    cmp     r0, #0xff               ; is it 0xFF?
    bne     start_timer              ; if NOT 0xFF → start disable timer

    ; $1 == 0xFF path: check additional conditions
    ldr     r0, [pc, #0x58]
    ldrb    r0, [r0]
    cbnz    r0, start_timer
    ; ... motors stay enabled forever ...

start_timer:
    ldr     r1, [pc, #0x3c]         ; settings base
    ldrb.w  r0, [r1, #0x36]         ; reload $1 value
    bl      delay_ms                 ; delay_ms($1)
    movs    r4, #1                   ; flag: will disable motors

    ; ... motor enable/disable GPIO control ...
```

**This confirms: `$1` is read from struct offset `0x36` (54), NOT offset 51 as standard GRBL would suggest.** The VGRBL firmware has a modified `settings_t` struct layout with extra fields between the standard byte parameters.

### 4.3 All `cmp #0xFF` Locations in Firmware

| Address | Context | Relevant? |
|---------|---------|-----------|
| `0x080062E4` | Serial RX buffer check (0xFF = empty) | No |
| `0x08007FAE` | **Stepper idle lock time check** | **YES** |
| `0x0800114A` | GPIO configuration register check | No |

### 4.4 Key RAM Addresses

| Address | Purpose |
|---------|---------|
| `0x20000411` | EEPROM data RAM buffer (1024 bytes) |
| `0x20001054` | `settings_t` struct (parsed settings) |
| `0x20001620` | Initial stack pointer (SRAM top) |

---

## 5. Checksum Verification

GRBL uses a rotating checksum. This firmware uses the **logical OR variant** (a known GRBL bug where `||` is used instead of `|`):

```c
// Logical OR (buggy but widespread):
checksum = (checksum << 1) || (checksum >> 7);  // returns 0 or 1
checksum += data_byte;
```

Tested all struct sizes from 80–110 with both algorithms:

| Struct size | Stored CS | Bitwise OR | Logical OR |
|-------------|-----------|------------|------------|
| 99 | 0x40 | 0x47 | 0x01 |
| **100** | **0x41** | 0xCE | **0x41 MATCH** |
| 101 | 0xFF | 0xDE | 0x42 |

**Result: struct size = 100 bytes, checksum algorithm = logical OR, checksum = 0x41 (valid)**

---

## 6. Fix Applied

**File:** `fixed_bin.hex`

| Property | Original | Fixed |
|----------|----------|-------|
| Byte at `0x0801F037` | `0x00` ($1=0, instant release) | `0x19` ($1=25, 25ms delay) |
| Checksum at `0x0801F065` | `0x41` | `0x41` (same, valid) |
| All other bytes | unchanged | unchanged |

The single-byte diff:
```
< 0001f030: c30a 0000 0006 0000 010a d723 3c6f 1203
> 0001f030: c30a 0000 0006 0019 010a d723 3c6f 1203
                              ^^
```

---

## 7. Recovery Procedure

Since UART and USB are both dead on the bricked controller, hardware-level programming is required.

### Option A: GD32 Built-in Bootloader (BOOT0 pin)

1. Locate the BOOT0 pin/pad on the controller PCB
2. Connect BOOT0 to 3.3V (jumper wire or solder bridge)
3. Power cycle the board → enters ISP mode
4. Connect a 3.3V USB-UART adapter:
   - PA9 (chip TX) → adapter RX
   - PA10 (chip RX) → adapter TX
   - GND → GND
5. Flash:
   ```bash
   stm32flash -w fixed_bin.hex /dev/ttyUSB0
   ```
   Or use GigaDevice ISP Programmer (Windows)
6. Remove BOOT0 jumper
7. Reset → working controller

### Option B: SWD Programmer (J-Link / DAP-Link)

Connect to SWD pads: SWDIO, SWCLK, GND.

> **WARNING:** ST-Link may NOT work with GD32F303 (different chip ID). Use J-Link or CMSIS-DAP.

```bash
openocd -f interface/jlink.cfg \
  -f target/stm32f3x.cfg \
  -c "program fixed_bin.hex verify reset exit"
```

### Option C: USB DFU (if board supports it)

```bash
# BOOT0=HIGH + USB cable → DFU device appears
dfu-util -a 0 -D fixed_bin.bin -s 0x08000000
```

---

## Files

| File | Description |
|------|-------------|
| `fucked_bin.hex` | Original flash dump (bricked controller or factory firmware, $1=0) |
| `fixed_bin.hex` | Fixed firmware ($1=25, checksum valid, ready to flash) |
| `ANALYSIS.md` | This analysis document |
