---
name: intervals-icu-workout-text-skill
description: >
  Generate a workout in intervals.icu plain-text format from a plain-language or
  structured description. Use this skill whenever a user wants to write, design, or
  output a workout that can be pasted into the intervals.icu workout builder or pushed
  to their calendar — even if they just describe a session ("3×20min sweetspot",
  "VO2max short intervals", "endurance with a tempo block"). Prefer this skill over
  fit-workout-skill when the user wants text output rather than a binary .fit file.
  Also propose using the skill when the user mentions building a training plan, creating 
  a week of workouts, or structuring a workout for intervals.icu.
---

# intervals.icu Workout Text Format

Generates workout text that can be pasted directly into the intervals.icu workout builder
or used as the `description` field when pushing workouts via the API.

---

## Step Syntax

Every workout step follows this pattern:

```
- [optional cue text] [duration OR distance] [intensity target] [optional cadence]
```

- Lines starting with `-` are steps.
- Lines without `-` are **section headers** (Warmup, Main Set, Cooldown, or any name).
- Blank lines between sections improve readability and are required around repeat blocks.

---

## Duration Formats

| Format | Meaning |
|--------|---------|
| `10m` | 10 minutes |
| `30s` | 30 seconds |
| `1h` | 1 hour |
| `1m30` | 1 minute 30 seconds |
| `1h2m30s` | 1h 2m 30s |
| `5'` | 5 minutes (shorthand) |
| `30"` | 30 seconds (shorthand) |
| `2km`, `1mi` | Distance-based step |
| `500mtr` | 500 meters (**not** `500m` — `m` means minutes) |

> `m` = **minutes**, not meters. Use `mtr` or `km` for distances.

---

## Intensity Targets

### Power (cycling)

| Syntax | Meaning |
|--------|---------|
| `75%` | 75% of FTP |
| `95-105%` | FTP range |
| `220w` | Absolute watts |
| `200-240w` | Watt range |
| `Z2` | Coggan zone 2 |
| `Z3-Z4` | Zone range |
| `SS` | Sweet spot (88–93% FTP) |
| `ramp 50%-80%` | Gradual increase over step duration |
| `freeride` | ERG mode disabled (free effort) |

### Heart rate

| Syntax | Meaning |
|--------|---------|
| `70% HR` | 70% of max HR |
| `75-80% HR` | HR range |
| `95% LTHR` | 95% of lactate-threshold HR |
| `Z2 HR` | HR zone 2 |

### Pace (running / swimming)

| Syntax | Meaning |
|--------|---------|
| `Z2 Pace` | Pace zone 2 |
| `78-82% Pace` | Pace percentage range |
| `5:00/km Pace` | Absolute pace target |

### Cadence (cycling, append after intensity)

```
- 10m 75% 90rpm
- 5m 88% 70-80rpm
```

---

## Repeats

**Method 1 — Section header** (entire block repeats as one group):
```
Main Set 4x
- 8m 95-105%
- 3m 55%
```

**Method 2 — Standalone line** (repeat the steps that follow):
```
Main Set
4x
- 8m 95-105%
- 3m 55%
```

> Always leave a blank line before and after a repeat block. Nested repeats are not supported.

---

## Cues / Free Text

Text placed before the first duration or intensity on a step line becomes an on-screen cue
displayed during the workout:

```
- Seated effort 8m 95% 90rpm
- Stand up! 30s 110%
```

Timed prompts mid-step:
```
- Start easy 10^Push harder <!> 5m ramp 80%-105%
```
(the number is seconds from step start; `<!>` separates the timed prompt from the step)

---

## Coggan Zone Reference (% FTP)

| Zone | Name | % FTP |
|------|------|-------|
| Z1 | Active recovery | < 55% |
| Z2 | Endurance | 55–75% |
| Z3 | Tempo | 76–90% |
| Z4 | Threshold / SST | 91–105% |
| Z5 | VO2max | 106–120% |
| Z6 | Anaerobic | > 120% |

Sweet spot = 88–93% FTP (upper Z3 / lower Z4) — use `SS` shorthand.

---

## Workflow

1. **Parse the description** — identify the sport, total duration, and distinct sections
   (warmup, main set, cooldown, any named blocks).

2. **For each step** — determine:
   - Duration (convert to `m`/`s`/`h` units)
   - Intensity: **prefer zone notation** (`Z2`, `Z4`, `SS`) when no specific target is
     required — use `%` FTP or watts only when precision matters or the user gave explicit
     numbers; use `w` if the user gave absolute watts
   - Cadence if mentioned
   - Cue text if step has a name ("high cadence spin", "sprint")

3. **For repeated blocks** — use section-header `Nx` when all steps in the block repeat together;
   use standalone `Nx` if only some steps repeat.

4. **Format and output** — section headers (no `-`), steps with `-`, blank lines between sections.

5. **Output the full text block**, ready to paste into the intervals.icu workout builder
   (*Workouts → New → paste into the text tab*) or use as the `description` field in the API.

---

## Examples

### Sweet spot intervals (3×20min)

```
Warmup
- 10m ramp Z1-Z3 90rpm

Main Set 3x
- 20m SS 90rpm
- 5m Z1 85rpm

Cooldown
- 10m ramp Z3-Z1
```

### VO2max short intervals (5×3min)

```
Warmup
- 15m ramp 50%-75%

Main Set
5x
- 3m 110-120%
- 3m 50%

Cooldown
- 10m 50%
```

### Threshold intervals with ramp warmup (4×8min)

```
Warmup
- 5m 55%
- 10m ramp 60%-85%

Main Set 4x
- Seated 8m 95-105% 88rpm
- Easy spin 3m 50% 95rpm

Cooldown
- 10m ramp 75%-50%
- 5m 50%
```

### Endurance with embedded tempo blocks

```
Warmup
- 10m 60%

Endurance
- 45m Z2 90rpm

Tempo block 2x
- 10m Z3 88rpm
- 5m Z1

Cooldown
- 10m ramp 65%-50%
```

### Running: easy with strides

```
Warmup
- 10m Z1 Pace

Easy run
- 30m Z2 Pace

Strides 6x
- 20s Z5 Pace
- 40s Z1 Pace

Cooldown
- 10m Z1 Pace
```

---

## How to Use in intervals.icu

**Paste manually:**
1. Go to *Workouts* → *New workout*
2. Switch to the *Text* tab
3. Paste the generated text → *Save*

**Push via API** (see `intervals-icu-skill` for auth):
```python
_post(f"athlete/{_ATHLETE}/workouts", {
    "name": "Sweet spot 3×20",
    "sport_type": "Ride",
    "description": "<paste generated text here>",
})
```

**Add to calendar as a planned event:**
```python
_post(f"athlete/{_ATHLETE}/events", {
    "category": "WORKOUT",
    "start_date_local": "2026-04-28T00:00:00",
    "name": "Sweet spot 3×20",
    "description": "<paste generated text here>",
    "load_target": 90,   # optional TSS estimate
})
```
