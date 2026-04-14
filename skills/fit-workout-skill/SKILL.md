---
name: fit-workout-skill
description: >
  Generate structured workout .fit files for Garmin devices and platforms like
  intervals.icu. Use this skill whenever a user wants to create, export, or convert
  a workout into a .fit file — even if they just describe a session in plain language
  like "4x10 tempo at 80% FTP" or "20min sweetspot intervals". Also trigger when the
  user mentions sending a workout to their Garmin, uploading to intervals.icu, or
  generating a structured training file. Supports cycling workouts with power targets
  (watts or % FTP). Always use this skill rather than ad-hoc scripting when .fit
  workout output is needed.
---

# FIT Workout Generator

Generates valid Garmin-compatible `.fit` workout files from structured or plain-language
workout descriptions. Files are compatible with intervals.icu import and USB transfer
to Garmin devices.

## Dependencies

```bash
pip install fitparse --break-system-packages   # for verification only
```

No external FIT writer library needed — files are built with raw `struct` packing
per the official Garmin FIT SDK spec.

---

## Key Facts

- FIT workout files use **little-endian** byte order
- Duration for TIME steps is stored in **milliseconds** (`seconds × 1000`)
- Power targets use a **+1000 offset** (e.g. 211W → stored as 1211)
- Field numbers must match the Garmin FIT SDK profile exactly (see below)
- CRC is computed for both the file header and the full file

### Correct workout_step field numbers (mesg 27)

| Field name              | Num | Type    | Size |
|-------------------------|-----|---------|------|
| message_index           | 254 | uint16  | 2    |
| wkt_step_name           | 0   | string  | 16   |
| duration_type           | 1   | uint8   | 1    |
| duration_value          | 2   | uint32  | 4    |
| target_type             | 3   | uint8   | 1    |
| target_value            | 4   | uint32  | 4    |
| custom_target_value_low | 5   | uint32  | 4    |
| custom_target_value_high| 6   | uint32  | 4    |
| intensity               | 7   | uint8   | 1    |

### Enum values

**duration_type**: `0` = TIME  
**target_type**: `4` = POWER  
**intensity**: `0` = active, `1` = rest, `2` = warmup, `3` = cooldown  
**sport** (workout mesg field 4): `2` = cycling  
**file type** (file_id mesg field 0): `5` = workout  

---

## Code Template

Copy and adapt this template. The generator script is self-contained.

```python
import struct, datetime

def fit_crc(data, crc=0):
    crc_table = [
        0x0000, 0xCC01, 0xD801, 0x1400, 0xF001, 0x3C00, 0x2800, 0xE401,
        0xA001, 0x6C00, 0x7800, 0xB401, 0x5000, 0x9C01, 0x8801, 0x4400,
    ]
    for byte in data:
        tmp = crc_table[crc & 0xF]
        crc = (crc >> 4) & 0x0FFF
        crc = crc ^ tmp ^ crc_table[byte & 0xF]
        tmp = crc_table[crc & 0xF]
        crc = (crc >> 4) & 0x0FFF
        crc = crc ^ tmp ^ crc_table[(byte >> 4) & 0xF]
    return crc

FIT_EPOCH = datetime.datetime(1989, 12, 31, 0, 0, 0, tzinfo=datetime.timezone.utc)
ts = int((datetime.datetime.now(datetime.timezone.utc) - FIT_EPOCH).total_seconds())

WARMUP, ACTIVE, REST, COOLDOWN = 2, 0, 1, 3

def str16(s):
    b = s.encode('utf-8')[:15]
    return b + b'\x00' * (16 - len(b))

def build_fit(workout_name, steps_data, filename):
    """
    steps_data: list of (name, duration_secs, low_w, high_w, intensity)
    """
    body = bytearray()

    # Local 0: file_id (mesg 0)
    body += struct.pack('<BBBHB', 0x40, 0, 0, 0, 3)
    body += struct.pack('<BBB', 4, 4, 0x86)   # time_created uint32
    body += struct.pack('<BBB', 1, 2, 0x84)   # manufacturer uint16
    body += struct.pack('<BBB', 0, 1, 0x00)   # type uint8
    body += struct.pack('<B', 0x00)
    body += struct.pack('<I', ts)
    body += struct.pack('<H', 1)               # manufacturer = Garmin
    body += struct.pack('<B', 5)               # type = workout

    # Local 1: workout (mesg 26)
    body += struct.pack('<BBBHB', 0x41, 0, 0, 26, 3)
    body += struct.pack('<BBB', 4,  1, 0x00)  # sport
    body += struct.pack('<BBB', 6,  2, 0x84)  # num_valid_steps
    body += struct.pack('<BBB', 8, 16, 0x07)  # wkt_name string[16]
    body += struct.pack('<B', 0x01)
    body += struct.pack('<B', 2)               # cycling
    body += struct.pack('<H', len(steps_data))
    body += str16(workout_name)

    # Local 2: workout_step (mesg 27) — field numbers are CRITICAL
    body += struct.pack('<BBBHB', 0x42, 0, 0, 27, 9)
    body += struct.pack('<BBB', 254,  2, 0x84)  # message_index
    body += struct.pack('<BBB',   0, 16, 0x07)  # wkt_step_name
    body += struct.pack('<BBB',   1,  1, 0x00)  # duration_type
    body += struct.pack('<BBB',   2,  4, 0x86)  # duration_value
    body += struct.pack('<BBB',   3,  1, 0x00)  # target_type
    body += struct.pack('<BBB',   4,  4, 0x86)  # target_value
    body += struct.pack('<BBB',   5,  4, 0x86)  # custom_target_value_low
    body += struct.pack('<BBB',   6,  4, 0x86)  # custom_target_value_high
    body += struct.pack('<BBB',   7,  1, 0x00)  # intensity

    for idx, (name, dur_s, low_w, high_w, intensity) in enumerate(steps_data):
        body += struct.pack('<B', 0x02)
        body += struct.pack('<H', idx)
        body += str16(name)
        body += struct.pack('<B', 0)               # duration_type = TIME
        body += struct.pack('<I', dur_s * 1000)    # ms — NOT raw seconds
        body += struct.pack('<B', 4)               # target_type = POWER
        body += struct.pack('<I', 0)               # target_value = 0 (custom)
        body += struct.pack('<I', low_w  + 1000)   # +1000 offset required
        body += struct.pack('<I', high_w + 1000)   # +1000 offset required
        body += struct.pack('<B', intensity)

    # Header + CRCs
    data_size = len(body)
    hdr = struct.pack('<BBHI4s', 14, 0x10, 0x083C, data_size, b'.FIT')
    hdr += struct.pack('<H', fit_crc(hdr))
    full = bytes(hdr) + bytes(body)
    full += struct.pack('<H', fit_crc(full))

    with open(filename, 'wb') as f:
        f.write(full)
```

---

## Workflow

1. **Parse the workout** from user input — plain language, a table, or a training plan doc.
   Convert all power targets to absolute watts using FTP if given as %.

2. **Build steps list**: each step is `(name, duration_secs, low_w, high_w, intensity)`.
   - Warm-ups and cool-downs → `WARMUP` / `COOLDOWN`
   - Work intervals → `ACTIVE`
   - Recovery → `REST`
   - Name strings are truncated to 15 chars + null terminator (16 bytes total)

3. **Generate the file** using the template above.

4. **Verify** by decoding with fitparse:
   ```bash
   pip install fitparse --break-system-packages
   python3 -c "
   from fitparse import FitFile
   for msg in FitFile('output.fit').get_messages('workout_step'):
       print({f.name: f.value for f in msg.fields})
   "
   ```
   Check that `duration_time` is in seconds (e.g. `900.0` for 15min),
   `target_type` is `'power'`, and `intensity` labels are correct.

5. **Copy to `/mnt/user-data/outputs/`** and present with `present_files`.

---

## Common Errors and Fixes

| Symptom | Cause | Fix |
|---|---|---|
| Steps don't appear in intervals.icu | Wrong field numbers in mesg 27 definition | Use exact field numbers from table above — this was the hard-won fix |
| Power targets show 0W or garbage | Missing +1000 offset on custom_target_value_low/high | Add 1000 to both low and high watt values |
| Duration shows as 0.9s instead of 900s | Duration stored in ms, fitparse divides by 1000 — this is correct | No fix needed; verify by checking `duration_time` field (should equal seconds) |
| File rejected as invalid | CRC mismatch | Ensure CRC is computed over the full header bytes including the 4-byte size field |
| Workout name truncated | str16() only keeps 15 chars | Expected — FIT spec limits step names to 16 bytes incl. null |

---

## Deployment

- **intervals.icu**: Import via Library → Workouts → Import (accepts `.fit` workout files directly)
- **Garmin USB**: Drop into `GARMIN/NEWFILES/` on the device; appears under Training → Workouts
- **Garmin Connect**: Does NOT support workout `.fit` import (only activity files) — use intervals.icu or USB instead
