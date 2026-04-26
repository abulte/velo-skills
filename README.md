# velo-skills

A collection of [Agent Skills](https://skills.sh) for cycling — training, equipment, and data tools for AI agents.

## Install

**Claude Code (CLI)**

```bash
npx skills add abulte/velo-skills
```

**Claude Desktop**

Claude Desktop does not support filesystem-based skills. Each skill must be uploaded manually as a ZIP file via Settings → Capabilities → Skills → Upload skill.

```fish
# build zips from the repo root
for skill in skills/*/
    zip -j (basename $skill).zip $skill/SKILL.md
end
```

## Skills

| Skill | Description |
|-------|-------------|
| [fit-workout-skill](skills/fit-workout-skill/SKILL.md) | Generate Garmin-compatible `.fit` workout files from plain-language or structured workout descriptions. Supports power targets (watts or % FTP), intervals.icu import, and USB transfer to Garmin devices. |
| [intervals-icu-skill](skills/intervals-icu-skill/SKILL.md) | Fetch, interpret, and reason over intervals.icu athlete data — activities, wellness, PMC metrics (CTL/ATL/TSB), power curves, events, and workouts. Also covers writing code against the intervals.icu API. |
| [intervals-icu-workout-text-skill](skills/intervals-icu-workout-text-skill/SKILL.md) | Generate intervals.icu plain-text workout format from a plain-language or structured description. Output can be pasted into the workout builder or pushed via the API. |
