---
name: food-diary
description: Log and evaluate food intake from images or notes. Use when the user attaches, pastes, or points to food photos, drink photos, receipts, menus, delivery screenshots, or asks to log food, update a food diary, process a meal, 整理饮食记录, 记一餐, 记录吃的, 评价饮食, generate a diet report, sync Apple Health, or process the legacy Kibble inbox. Default food logging accepts direct chat images and local image paths without requiring the Kibble app or inbox. It writes nutrition JSON and index.csv entries to the configured health-log repo, can optionally process the old inbox, can sync Apple Health exports, and can generate Word diet/health reports.
---

# Food Diary

This skill logs meals from **direct user-provided images first**. The user can attach an image in chat, paste a screenshot, provide a local image path, or describe a meal in text. The old Kibble app inbox remains a legacy input path only when the user explicitly asks to process it.

The skill stores source meal records in a health-log repo, usually `/Users/tom/Desktop/health-log`, using the existing `index.csv` plus per-item `.nutrition.json` and optional `.note.txt` files. It does not need the Kibble desktop app to receive new food images.

## Modes

1. **Log food** (default) - process direct chat images, pasted screenshots, local image paths, or meal notes into the food diary.
2. **Legacy inbox process** - drain `<repo>/inbox/` only when the user explicitly says to process the Kibble inbox.
3. **Apple Health sync** - parse `~/Desktop/apple_health_export/` into `<repo>/health/`.
4. **Diet report** - aggregate `index.csv` into a Word report on the Desktop.
5. **Health report** - after Apple Health sync, generate a Word health report on the Desktop.

Decide the mode from the user request. If the request contains food imagery or meal text and does not mention `inbox` or `Kibble`, use **Log food**.

After a food logging run finishes, ask the user: `also sync Apple Health export? (y/n)`. If yes, run Apple Health sync and fold it into the same commit.

Diet report is standalone and read-only on the repo. It does not touch the inbox, does not sync Apple Health, and does not commit.

## Repo And Config

Resolve the repo path in this order:

1. A repo path explicitly supplied by the user.
2. `repo_path` from `~/Library/Application Support/kibble/config.toml`, if present.
3. `/Users/tom/Desktop/health-log`.

Also read `[meal_times]` from the same config when present:

- `breakfast = ["06:00", "09:59"]`
- `lunch = ["10:00", "13:59"]`
- `snack = ["14:00", "16:59"]`
- `dinner = ["17:00", "19:59"]`
- anything else is `late_night`

Abort clearly if the repo path is missing or does not exist.

## Inputs

Supported direct inputs:

- Attached chat images or pasted screenshots.
- Local image paths supplied by the user.
- Food descriptions in text, with or without time/date.
- Optional notes in the same message, such as "昨天晚饭", "lunch", or "shared with two people".

Supported legacy inputs:

- `<repo>/inbox/` images plus sibling `*.note.txt` files, only when explicitly requested.

For attached chat images with no accessible local file path, use the visible image content directly. Do not ask the user to drag the image into Kibble or copy it into `inbox/`.

## Core Rules

1. **Do not invent food labels, capture times, calories, macros, or vendors.** Every value must trace to user text, image content, file metadata, current submission context, or a WebSearch/source URL. If a value cannot be determined, write `null` and add a note.
2. **Direct inputs are non-destructive.** Never delete or move user-supplied local images. For chat attachments, do not try to persist original image bytes unless the user explicitly asks.
3. **Legacy inbox is the only destructive path.** In legacy inbox mode, preserve old behavior: write text artefacts, then delete the inbox image after successful logging.
4. **Text artefacts are the durable record.** Store `.nutrition.json`, optional `.note.txt`, and `index.csv`; do not store photos by default.
5. **Idempotent on collisions.** Destination filename collision -> append `_1`, `_2`, etc. Apply the same suffix to matching note and nutrition files.
6. **One commit per run.** Food logging commit message: `skill: log N food item(s)`. Legacy inbox may use `skill: process N item(s)`.
7. **Honest confidence.** Every `.nutrition.json` must include `confidence: "high" | "med" | "low"`. If unsure between two levels, choose the lower one.
8. **Use exact dates.** When the user says relative dates such as "today", "yesterday", or "last night", resolve them using the current runtime date and timezone, then record the exact date.

## Food Logging Workflow

### Step 1 - Collect Items

Build a work list from the user's direct inputs:

- One entry per attached/pasted image.
- One entry per local image path.
- One entry for a text-only meal description if no image is supplied.

For each entry, carry:

- `source_kind`: `chat_image`, `local_image`, or `text`
- `source_name`: original filename if known, local basename if path supplied, otherwise `chat-YYYYMMDD-HHMMSS-N`
- `user_note`: the user's surrounding message or sibling note text
- `image_content`: the visible image if available

If no food image, receipt, menu, or meal note is present, say there is nothing to log.

### Step 2 - Determine Date, Time, And Meal Type

Use this priority order:

1. Explicit date/time from the user's message.
2. Image metadata for local image paths (`sips -g creation`, then `stat -f %Sm -t '%Y:%m:%d %H:%M:%S'`).
3. Current submission date/time from the runtime context.

For attached chat images, EXIF is often unavailable. In that case, use current submission date/time and add a note such as `"capture time unavailable; used chat submission time"`.

Classify meal type from the configured meal windows. If the user explicitly says "breakfast", "lunch", "dinner", "snack", "夜宵", etc., prefer that over the clock-derived category and note the override if useful.

### Step 3 - Image Type Detection

Classify each item into exactly one:

| `image_type` | Meaning | Nutrition source |
|---|---|---|
| `food` | Food or drink itself | Vision portion estimate + nutrition source per serving or per 100g |
| `receipt` | Printed/digital order receipt with line items | OCR line items + vendor nutrition lookup |
| `menu` | Menu or ordering screen | Use selected item only if visible or stated |
| `non_food` | Not food, receipt, or menu | No nutrition |
| `text` | Text-only meal description | User text + nutrition lookup |
| `unknown` | Image could not be read/classified | Stub record only |

Set `food_label`:

- `food` or `text`: 1-4 word lowercase English label, e.g. `beef noodles`, `mixed salad`, `iced latte`.
- Multiple visible items: join with `+`, e.g. `noodles+egg`.
- `receipt` / `menu`: `<vendor-slug> receipt` or `<vendor-slug> menu`.
- `non_food` or `unknown`: `unidentified`.

For local HEIC/HEIF files, create a transient preview only if needed:

```bash
sips -s format jpeg "<path>" --out "/tmp/food-diary-view-<random>.jpg"
```

If `sips` output is black/blank/unreadable, request approval to generate a Quick Look thumbnail:

```bash
mkdir -p /tmp/food-diary-ql
qlmanage -t -s 1600 -o /tmp/food-diary-ql "<path>"
```

Delete transient previews after use. Do not delete the source image unless in legacy inbox mode.

### Step 4 - Nutrition Lookup

For `food`:

1. Identify each distinct food/drink visible.
2. Estimate portion from visible cues. If no reliable cue exists, set `estimated_g: null` and downgrade confidence.
3. WebSearch `"<food name> calories per 100g"` or `"<food name> nutrition per serving"`.
4. Prefer USDA FoodData Central, official restaurant nutrition pages, package labels visible in the image, or reputable nutrition databases.
5. Compute calories/protein/carbs/fat from the selected serving or per-100g value. Round to integers.

For `receipt`:

1. OCR vendor and line items from the image.
2. Capture source-language item names, quantity, and price if visible.
3. WebSearch `"<vendor> <item name> calories"`.
4. Prefer the vendor's official nutrition page.
5. Multi-quantity rules: a size like `5 pc` or `5块` is not an extra multiplier; only multiply by the ordered quantity.

For `menu`:

- If a selected item is visible or stated in `user_note`, treat it like a receipt for that item.
- Otherwise write `items: []`, `confidence: low`, and note `"menu image without selected item"`.

For `text`:

- Use the user's description as the source item list.
- If quantity is vague, ask a concise follow-up only when the ambiguity would materially change the log. Otherwise log low confidence with the uncertainty in `notes`.

For `non_food`:

- Write an empty nutrition record with `confidence: high` and note `"image is not food, receipt, or menu"`.

For `unknown`:

- Write an empty nutrition record with `confidence: low` and note why the item could not be read or classified.

If WebSearch is unavailable, do not block the whole run. Use `source_url: null`, set or keep `confidence: low`, and add `"WebSearch unavailable"` to `notes`.

## Nutrition JSON

Write one pretty-printed UTF-8 JSON file with a trailing newline:

```json
{
  "image_type": "food | receipt | menu | non_food | text | unknown",
  "food_label": "string",
  "vendor": "string | null",
  "confidence": "high | med | low",
  "items": [
    {
      "name": "string (canonical English)",
      "name_source": "string | null",
      "quantity": 1,
      "estimated_g": 250,
      "calories": 420,
      "protein_g": 18,
      "carb_g": 55,
      "fat_g": 12,
      "source_url": "https://..."
    }
  ],
  "totals": {
    "calories": 420,
    "protein_g": 18,
    "carb_g": 55,
    "fat_g": 12
  },
  "notes": ["free-form caveats, units, exclusions"]
}
```

Rules:

- `totals.*` must equal the sum of `items[*].*` for calories/protein/carb/fat.
- Use integers for macros and calories, or `null` when unknown.
- `vendor` is `null` for `food`, `text`, `non_food`, and `unknown`.
- `vendor` is required for `receipt` and selected-item `menu` records when visible.
- `estimated_g` may be `null` for receipt/vendor serving entries.
- `source_url` is `null` only when no source was available; downgrade confidence and explain in `notes`.

## Filing

Destination layout:

```text
<repo>/YYYY/MM/DD/<meal_type>/<source_name>.nutrition.json
<repo>/YYYY/MM/DD/<meal_type>/<source_name>.note.txt   # only when user_note is non-empty
```

For direct chat images and local image paths, write artefacts directly to the destination folder and leave the original image untouched.

For legacy inbox mode only:

```text
inbox/IMG_xxx.HEIC.note.txt        -> YYYY/MM/DD/<meal>/IMG_xxx.HEIC.note.txt
inbox/IMG_xxx.HEIC.nutrition.json  -> YYYY/MM/DD/<meal>/IMG_xxx.HEIC.nutrition.json
inbox/IMG_xxx.HEIC                 -> deleted after successful logging
```

If an inbox image is unreadable after both `sips` and Quick Look, move it to `<repo>/unreadable/` with its note and write a low-confidence stub in the dated folder. Do not delete unreadable images.

## Update index.csv

Schema:

```text
date,time,meal_type,image_type,food_label,calories,protein_g,carb_g,fat_g,confidence,filename,note,path
```

Rules:

- Header is written only if the file does not exist.
- Numeric fields are integers or empty string for `null`.
- Quote fields according to RFC 4180.
- `filename` is the original filename or synthetic `chat-...` name.
- `note` is the user's original note/message, not generated item details.
- `path` is relative to `<repo>` and points to the `.nutrition.json` record.
- Use a temp file + rename to avoid partial writes.

## Commit And Push

Run from the repo:

```bash
git add -A
if ! git diff --cached --quiet; then
  git commit -m "<message>"
  git push
fi
```

Commit messages:

- Food logging: `skill: log N food item(s)`
- Legacy inbox: `skill: process N item(s)`
- Process + health: `skill: log N food item(s) + sync apple health`
- Health only: `skill: sync apple health (<date_range>)`

Push failures should be surfaced verbatim. Do not roll back local work.

## Summary

Print a concise table:

```text
logged N items, K errors

12:25 lunch    food     beef noodles       ~650 kcal (med)    IMG_1728.HEIC
18:22 dinner   receipt  mcdonalds receipt  ~1115 kcal (high)  chat-20260705-182233-1
                          ↳ filet-o-fish, hash brown, mcnuggets 5pc, cola zero (M)

-> pushed as commit <sha>
```

Use `≈` before calories for low-confidence records. For `non_food`, show `<filename>: skipped (non-food)`. For `unknown`, show `???` in derived columns.

## Apple Health Sync

This mode imports an iOS Health export into `<repo>/health/`.

Default path: `~/Desktop/apple_health_export/`. Inside, look for `导出.xml` first, then `export.xml`.

Run:

```bash
python3 ~/.codex/skills/food-diary/parse_apple_health.py \
  ~/Desktop/apple_health_export \
  <repo_path>
```

The script writes:

- `health/daily.csv`
- `health/workouts.csv`
- `health/ecg/*.csv`
- `health/raw_record_counts.md`
- `health/README.md`

It deliberately excludes:

- `workout-routes/*.gpx`
- `export_cda.xml`
- the raw `导出.xml` / `export.xml`

After successful sync, run:

```bash
python3 ~/.codex/skills/food-diary/generate_report.py <repo_path>
```

This writes a Word report to the Desktop:

```text
~/Desktop/food-diary-health-report-<YYYY-MM-DD>.docx
```

Reports and charts are derived artefacts. They must stay on Desktop and must not be committed to the repo.

## Diet Report

Diet report reads `<repo>/index.csv` and writes a Word report to the Desktop. It does not modify the repo and does not commit.

Run:

```bash
python3 ~/.codex/skills/food-diary/generate_diet_report.py <repo_path>
```

Output:

```text
~/Desktop/food-diary-report-<YYYY-MM-DD>.docx
```

The report includes:

- Overview: days logged, complete vs partial days, breakfast coverage, fast-food and sweet/fried counts.
- Per-day macro table.
- Trend charts for calories, fat, and protein with threshold lines.
- Data-driven signals where every bullet references a real number.

Charts plot only complete days (at least two meal slots or at least two items) to avoid treating partial logs as full-day nutrition.

If dependencies are missing, tell the user:

```bash
pip3 install python-docx matplotlib pandas
```

Then skip report generation rather than blocking food logging.

## Confidence Rubric

| Source | Confidence |
|---|---|
| Chain restaurant receipt, all items found on vendor official nutrition page | `high` |
| Chain restaurant receipt, some items from third-party database | `med` |
| Packaged food label visible and readable | `high` |
| Generic food with reputable per-100g source and reliable visual reference | `med` |
| Home-cooked/generic food with no clear portion reference | `low` |
| Multi-component dish where ingredients cannot be separated cleanly | `low` |
| Text-only vague meal description | `low` |
| Used own knowledge without source URL | `low` |

## Edge Cases

- Attached image has no filename -> use `chat-YYYYMMDD-HHMMSS-N`.
- Attached image has no metadata -> use current submission time and note it.
- Local image path exists but cannot be opened -> write `unknown` only if enough context exists; otherwise ask for a re-upload.
- Note but no image -> process as `text` if it describes food; otherwise say nothing to log.
- Multiple people shared the meal -> only divide nutrition if the user clearly says their share.
- Beverages with no calories, such as water or cola zero, may legitimately be 0.
- Cross-day wording like "昨晚" must be resolved to an exact date.
- Legacy inbox `.gitkeep` -> ignore.
- Legacy orphan `.note.txt` with no image -> move to `<repo>/orphans/` and do not delete.

## What This Skill Does Not Do

- Require the Kibble app for new images.
- Store original food photos by default.
- Delete or move direct user-provided files.
- Re-classify already-filed items into different meal folders unless the user explicitly asks.
- Add generic health advice not tied to logged data.
