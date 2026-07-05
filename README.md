# Food Diary

Food Diary is a Codex skill for turning meal photos, receipts, menus, and notes into a structured food diary.

The skill lives in [`skill/SKILL.md`](skill/SKILL.md) and is named `food-diary`. An agent can browse this public repo, read the skill definition, and install it locally (for example by copying or symlinking `skill/` into its skills directory).

## What It Can Do

- Log direct chat images, pasted screenshots, local image paths, and text-only meal notes.
- Classify entries as food, receipt, menu, non-food, text, or unreadable.
- Resolve date, time, and meal type from user text, image metadata, or submission time.
- Estimate portions, look up calories/macros, and write per-entry `.nutrition.json` files.
- Update `index.csv` with calories, protein, carbs, fat, confidence, notes, and record paths.
- Commit and push the resulting food diary records to the configured health-log repo.
- Optionally process a legacy `inbox/` folder when the user explicitly asks.
- Sync Apple Health exports and generate Word diet/health reports via the bundled Python scripts.

Vision, portion estimation, and nutrition lookup are performed by the agent running the skill — not by local computer-vision libraries. The skill provides the recording framework, file layout, and commit workflow.

## Install

Point your agent at this repo URL. After reading `skill/SKILL.md`, install the skill into your agent's skills directory. A typical layout:

```bash
# example: Codex skills directory
ln -s /path/to/food-diary/skill ~/.codex/skills/food-diary
```

Report scripts need Python dependencies:

```bash
pip3 install python-docx matplotlib pandas
```

## Reports

Food Diary can generate two Word reports on the Desktop:

- `food-diary-report-YYYY-MM-DD.docx` — diet report from `index.csv`, including daily macro tables, complete-day trend charts, threshold markers, and data-driven diet signals.
- `food-diary-health-report-YYYY-MM-DD.docx` — health report after Apple Health sync, combining Apple Health aggregates with the food diary log.

Apple Health sync imports a local iOS Health export into `health/daily.csv`, `health/workouts.csv`, and `health/ecg/*.csv`, while deliberately excluding GPS route files.

## Data Repo

Food Diary writes source records into a separate private health-log repo, usually configured by `repo_path`:

```text
health-log/
├── index.csv
├── YYYY/MM/DD/<meal_type>/*.nutrition.json
├── YYYY/MM/DD/<meal_type>/*.note.txt
├── health/daily.csv
├── health/workouts.csv
└── inbox/                  # legacy optional input
```

The repo stores source data only. Generated Word reports and charts stay on the Desktop and are not committed.

## Config

The skill resolves the data repo path in this order:

1. A path explicitly supplied by the user.
2. `repo_path` from `~/Library/Application Support/food-diary/config.toml`, if present.
3. `repo_path` from legacy `~/Library/Application Support/kibble/config.toml`, if present.
4. `/Users/tom/Desktop/health-log`.

## License

MIT
